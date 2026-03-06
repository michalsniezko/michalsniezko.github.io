# Scaling: CloudWatch, Autoscaling & Lambda Patterns

## SQS-Based Autoscaling

**Metric:** `ApproximateNumberOfMessagesVisible` — the number of messages in the queue waiting to be processed. More useful than CPU for worker-based architectures because a queue can be deep while workers sit idle waiting on I/O (low CPU, high backlog).

CPU-based scaling misses the actual problem: unprocessed work. A worker doing HTTP calls to a slow upstream might use 5% CPU while 50,000 messages pile up. SQS depth tells you how much work exists, not how hard the machines are working.

### Custom Metric: Backlog Per Instance

Raw queue depth doesn't account for fleet size. 10,000 messages with 2 workers is a problem; 10,000 messages with 100 workers is fine. The useful metric is **backlog per instance**.

```
BacklogPerInstance = ApproximateNumberOfMessagesVisible / RunningTaskCount
```

### Terraform: ECS Autoscaling on Queue Depth

```hcl
resource "aws_appautoscaling_target" "workers" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.worker.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "scale_on_queue" {
  name               = "worker-queue-depth-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.workers.resource_id
  scalable_dimension = aws_appautoscaling_target.workers.scalable_dimension
  service_namespace  = aws_appautoscaling_target.workers.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 100  # aim for ~100 messages per worker

    customized_metric_specification {
      metric_name = "BacklogPerInstance"
      namespace   = "Custom/WorkerScaling"
      statistic   = "Average"
      unit        = "Count"
    }

    scale_in_cooldown  = 300  # wait 5 min before scaling down
    scale_out_cooldown = 60   # scale up quickly
  }
}
```

### Publishing the Custom Metric (Lambda or Cron)

```bash
#!/bin/bash
QUEUE_URL="https://sqs.eu-west-1.amazonaws.com/123456789012/order-processing"
CLUSTER="production"
SERVICE="order-worker"

MSG_COUNT=$(aws sqs get-queue-attributes \
    --queue-url "$QUEUE_URL" \
    --attribute-names ApproximateNumberOfMessagesVisible \
    --query 'Attributes.ApproximateNumberOfMessagesVisible' \
    --output text)

TASK_COUNT=$(aws ecs describe-services \
    --cluster "$CLUSTER" \
    --services "$SERVICE" \
    --query 'services[0].runningCount' \
    --output text)

# Avoid division by zero
if [ "$TASK_COUNT" -eq 0 ]; then
    BACKLOG="$MSG_COUNT"
else
    BACKLOG=$((MSG_COUNT / TASK_COUNT))
fi

aws cloudwatch put-metric-data \
    --namespace "Custom/WorkerScaling" \
    --metric-name "BacklogPerInstance" \
    --value "$BACKLOG" \
    --unit Count
```

> **Cost/Performance Note:** Set `scale_in_cooldown` significantly higher than `scale_out_cooldown`. Scaling out is cheap (handle the spike). Scaling in too fast causes **flapping**: traffic spike → scale to 20 → spike subsides → scale to 2 → next batch arrives → scale to 20. A 5-minute scale-in cooldown lets transient dips pass without thrashing. Also: ECS tasks take 30–90 seconds to become healthy — factor this startup lag into your target value.

---

## CloudWatch Alarms for SQS Backlog

**Metric:** `ApproximateNumberOfMessagesVisible` (for backlog) and `ApproximateAgeOfOldestMessage` (for staleness). Age is often more actionable — 1,000 messages that are 2 seconds old is fine; 10 messages that are 30 minutes old means your consumer is dead.

### Terraform: Queue Depth Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "queue_backlog_high" {
  alarm_name          = "order-queue-backlog-critical"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3          # must breach 3 consecutive periods
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60         # 1-minute intervals
  statistic           = "Average"
  threshold           = 5000
  alarm_description   = "Order processing queue has > 5000 pending messages for 3+ minutes"

  dimensions = {
    QueueName = "order-processing"
  }

  alarm_actions = [aws_sns_topic.infra_alerts.arn]
  ok_actions    = [aws_sns_topic.infra_alerts.arn]

  tags = { Team = "backend", Severity = "critical" }
}
```

### Terraform: Message Age Alarm (Consumer Health)

```hcl
resource "aws_cloudwatch_metric_alarm" "queue_age_high" {
  alarm_name          = "order-queue-oldest-message-stale"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ApproximateAgeOfOldestMessage"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Maximum"
  threshold           = 600        # 10 minutes
  alarm_description   = "Oldest message in queue is > 10 minutes old — consumer may be stuck"

  dimensions = {
    QueueName = "order-processing"
  }

  alarm_actions = [aws_sns_topic.infra_alerts.arn]
}
```

### Alert Routing (SNS → Slack/PagerDuty)

```hcl
resource "aws_sns_topic" "infra_alerts" {
  name = "infra-alerts"
}

resource "aws_sns_topic_subscription" "slack_webhook" {
  topic_arn = aws_sns_topic.infra_alerts.arn
  protocol  = "https"
  endpoint  = "https://hooks.slack.com/services/T00/B00/xxxxx"
}
```

> **Cost/Performance Note:** `evaluation_periods = 3` with `period = 60` means the alarm triggers after 3 consecutive minutes above threshold. This prevents alert noise from brief spikes (e.g., a batch job drops 10k messages, workers drain it in 90 seconds). For `ApproximateAgeOfOldestMessage`, use `statistic = "Maximum"` — you care about the worst case, not the average.

---

## Lambda Monitoring with CloudWatch

**Metrics to watch:**

| Metric              | What It Tells You                                      | Alert When                     |
|---------------------|--------------------------------------------------------|--------------------------------|
| `Duration`          | Execution time per invocation                          | p99 > 80% of timeout setting   |
| `Errors`            | Unhandled exceptions                                   | Any sustained errors           |
| `Throttles`         | Invocations rejected (concurrency limit hit)           | > 0 for critical Lambdas       |
| `ConcurrentExecutions` | How many instances running simultaneously           | Approaching account/function limit |
| `IteratorAge`       | How far behind the stream consumer is (SQS/Kinesis)    | Growing steadily               |

### Terraform: Duration Alarm (Timeout Early Warning)

```hcl
resource "aws_cloudwatch_metric_alarm" "lambda_duration_high" {
  alarm_name          = "order-publisher-duration-p99"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "Duration"
  namespace           = "AWS/Lambda"
  period              = 300
  extended_statistic  = "p99"
  threshold           = 8000       # 8 seconds (timeout is 10s)
  alarm_description   = "Lambda p99 duration approaching timeout"

  dimensions = {
    FunctionName = "order-publisher"
  }

  alarm_actions = [aws_sns_topic.infra_alerts.arn]
}
```

### Structured Logging for CloudWatch Insights

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    start = time.time()

    # ... business logic ...

    duration_ms = (time.time() - start) * 1000
    remaining_ms = context.get_remaining_time_in_millis()

    logger.info(json.dumps({
        "event": "invocation_complete",
        "function": context.function_name,
        "request_id": context.aws_request_id,
        "duration_ms": round(duration_ms, 2),
        "remaining_ms": remaining_ms,
        "memory_limit_mb": context.memory_limit_in_mb,
        "records_processed": len(event.get("Records", [])),
    }))
```

### CloudWatch Insights Query

```
fields @timestamp, @requestId, duration_ms, remaining_ms, records_processed
| filter event = "invocation_complete"
| stats avg(duration_ms) as avg_ms,
        max(duration_ms) as max_ms,
        percentile(duration_ms, 99) as p99_ms
  by bin(5m)
| sort @timestamp desc
```

> **Cost/Performance Note:** CloudWatch Logs charges per GB ingested. A Lambda processing 1M events/day with verbose logging can generate 10+ GB/month of logs (~$5/GB). Log at `INFO` in production, never `DEBUG`. Use structured JSON logging so you can query with CloudWatch Insights instead of dumping everything and grepping later. Set a log retention policy (14–30 days) — unset retention means logs live forever and costs grow silently.

---

## AWS SDK Pitfalls in High-Concurrency Environments

**Scenario:** Your Lambda processes SQS batches of 10 records. Each record triggers 3 AWS API calls (DynamoDB get, S3 put, SNS publish). At 100 concurrent Lambda invocations, that's 3,000 near-simultaneous API calls. The SDK's default connection handling wasn't designed for this.

### Common Issues

| Problem                       | Symptom                                        | Fix                                         |
|-------------------------------|------------------------------------------------|---------------------------------------------|
| Connection exhaustion         | `TooManyRequestsException` or socket timeouts  | Reuse SDK client outside the handler        |
| Cold start latency            | First invocation takes 3–5 seconds             | Initialize clients at module level          |
| Memory leak (PHP/Node)        | Lambda memory grows until OOM kill             | Don't create new clients per invocation     |
| Retry storm                   | Cascading failures under load                  | Configure max retries + exponential backoff |

### Client Reuse (Module-Level Init)

```python
import boto3

# Initialized ONCE per Lambda container — persists across warm invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('orders')
sns = boto3.client('sns')

def handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        # Reuses the existing connection pool
        table.put_item(Item=body)
        sns.publish(TopicArn=TOPIC_ARN, Message=record['body'])
```

### PHP SDK: Retry Configuration

```php
use Aws\Sdk;

$sdk = new Sdk([
    'region'  => 'eu-west-1',
    'version' => 'latest',
    'retries' => [
        'mode'        => 'adaptive',  // token-bucket based backoff
        'max_attempts' => 3,
    ],
    'http' => [
        'connect_timeout' => 2,
        'timeout'         => 5,
    ],
]);

// Create clients from the shared SDK instance
$sqs = $sdk->createSqs();
$sns = $sdk->createSns();
```

### Lambda Concurrency Guardrails

```hcl
resource "aws_lambda_function" "order_processor" {
  function_name = "order-processor"
  # ...

  reserved_concurrent_executions = 50  # cap at 50 simultaneous invocations
}
```

> **Cost/Performance Note:** `reserved_concurrent_executions` protects downstream services (a database that can handle 50 connections, not 500). But it also means excess invocations get **throttled** — messages return to the SQS queue and retry. Set `reserved_concurrent_executions` based on what your downstream dependencies can handle, not what Lambda can scale to. Monitor the `Throttles` metric to confirm you're not silently dropping work.

---

## Lambda Code Sharing: Multi-Handler vs. Monolith Router

**Scenario:** You have 5 event sources (API Gateway, SQS, S3, EventBridge, SNS) that share 80% of their logic (validation, data access, SNS publishing). Do you deploy 5 separate Lambda functions from the same codebase, or one Lambda with an internal router?

### Option A: Multiple Functions, Different Handlers (Recommended)

```
shared-codebase/
├── handlers/
│   ├── api.py           # API Gateway → handler
│   ├── sqs_consumer.py  # SQS → handler
│   ├── s3_trigger.py    # S3 → handler
│   └── scheduler.py     # EventBridge → handler
├── core/
│   ├── service.py       # Shared business logic
│   └── publisher.py     # Shared SNS publish
└── template.yaml        # SAM: 4 functions, 1 codebase
```

```yaml
# template.yaml
Resources:
  ApiHandler:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/api.handler
      MemorySize: 256
      Timeout: 10
      Events:
        Api:
          Type: Api
          Properties: { Path: /orders, Method: post }

  SqsConsumer:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/sqs_consumer.handler
      MemorySize: 512        # needs more memory for batch processing
      Timeout: 60            # SQS batches take longer
      ReservedConcurrentExecutions: 30
      Events:
        Queue:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10
```

### Option B: Monolith Router (Anti-Pattern at Scale)

```python
# handler.py — single entry point for everything
def handler(event, context):
    if "httpMethod" in event:
        return handle_api(event)
    elif "Records" in event and event["Records"][0].get("eventSource") == "aws:sqs":
        return handle_sqs(event)
    elif "Records" in event and event["Records"][0].get("eventSource") == "aws:s3":
        return handle_s3(event)
    else:
        raise ValueError(f"Unknown event source: {json.dumps(event)[:200]}")
```

### Comparison

| Aspect                    | Multi-Handler                      | Monolith Router                    |
|---------------------------|------------------------------------|------------------------------------|
| Memory/Timeout tuning     | Per function — optimized           | One size fits all — wasteful       |
| Concurrency limits        | Per function — isolated            | Shared — SQS spike starves API    |
| Error blast radius        | Bug in S3 handler doesn't affect API | Bug in router breaks everything |
| Cold starts               | Each function has its own cold start pool | Single pool, fewer cold starts |
| Deployment                | Can deploy one function independently | All-or-nothing deploy            |
| Monitoring                | Metrics per function — clear       | One set of metrics — noisy        |

> **Cost/Performance Note:** The monolith router's only real advantage is fewer cold starts (one warm pool serves all events). But this is offset by the inability to right-size memory and timeout per event type. An API handler needs 256MB and 10s; an SQS batch processor needs 512MB and 60s. With the monolith, you set 512MB/60s for everything — overpaying for API calls and under-provisioning nothing. At scale, the wasted memory across millions of API invocations far exceeds the cold start savings.

---

## Routing via SNS Subscriptions

**Metric:** No custom metric needed — routing is declarative via subscription configuration. Monitor `NumberOfMessagesPublished` (SNS) and `NumberOfMessagesReceived` (SQS) to verify messages flow.

**Scenario:** Your order service publishes `order.created`, `order.paid`, and `order.cancelled` events. The billing service only needs `order.paid`. The analytics service needs everything. The notification service needs `order.paid` and `order.cancelled`. Implementing this routing in application code means every publisher needs to know about every consumer.

**Solution:** Publish everything to one SNS topic. Routing lives in subscription filter policies — the publisher is completely unaware of who consumes what.

### Architecture

```
                                 Filter: order_status=["paid"]
Order Service ──► SNS Topic ──┬──────────────────────────────► SQS (Billing)
                              │
                              │  Filter: (none — receives all)
                              ├──────────────────────────────► SQS (Analytics)
                              │
                              │  Filter: order_status=["paid","cancelled"]
                              └──────────────────────────────► SQS (Notifications)
```

### Terraform: Topic + Subscriptions with Filters

```hcl
resource "aws_sns_topic" "order_events" {
  name = "order-events"
}

resource "aws_sqs_queue" "billing" {
  name = "billing-order-events"
}

resource "aws_sqs_queue" "analytics" {
  name = "analytics-order-events"
}

resource "aws_sqs_queue" "notifications" {
  name = "notifications-order-events"
}

# Billing: only paid orders
resource "aws_sns_topic_subscription" "billing" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.billing.arn

  raw_message_delivery = true
  filter_policy = jsonencode({
    order_status = ["paid"]
  })
}

# Analytics: everything (no filter)
resource "aws_sns_topic_subscription" "analytics" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.analytics.arn

  raw_message_delivery = true
  # No filter_policy = receives all messages
}

# Notifications: paid + cancelled
resource "aws_sns_topic_subscription" "notifications" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.notifications.arn

  raw_message_delivery = true
  filter_policy = jsonencode({
    order_status = ["paid", "cancelled"]
  })
}
```

### Publisher (Unchanged Regardless of Consumers)

```python
sns.publish(
    TopicArn=TOPIC_ARN,
    Message=json.dumps({"orderId": "ord-456", "amount": 199.99}),
    MessageAttributes={
        "order_status": {"DataType": "String", "StringValue": "paid"},
        "region":       {"DataType": "String", "StringValue": "eu-west"},
    }
)
```

Adding a new consumer (e.g., a fraud detection service) requires zero changes to the publisher — just add a new SQS queue and SNS subscription with the appropriate filter.

### Verifying Routing (CLI)

```bash
# List all subscriptions on a topic
aws sns list-subscriptions-by-topic \
    --topic-arn arn:aws:sns:eu-west-1:123456789012:order-events

# Check a specific subscription's filter policy
aws sns get-subscription-attributes \
    --subscription-arn arn:aws:sns:eu-west-1:123456789012:order-events:a1b2c3d4 \
    --query 'Attributes.FilterPolicy' \
    --output text
```

> **Cost/Performance Note:** SNS charges per publish ($0.50/million requests), not per subscription delivery. But each SQS queue that receives the message incurs SQS costs. An unfiltered subscription on a high-volume topic (10M messages/day) where the consumer discards 95% of messages wastes ~$3.80/day in SQS receive + delete API calls alone. Always apply the narrowest filter policy possible — you're paying for every message that lands in a queue, even if the consumer immediately discards it.
