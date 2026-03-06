---
layout: default
title: AWS Event-Driven Architecture: SNS + SQS Patterns
nav_order: 2
---

# AWS Event-Driven Architecture: SNS + SQS Patterns

## SNS Fan-out to SQS

**Problem:** Service A needs to notify Services B, C, and D when an order is placed. Direct HTTP calls create tight coupling — if Service C is down, Service A either blocks or needs retry logic for every downstream consumer.

**Pattern:** Publish once to an SNS topic. Each downstream service owns its own SQS queue subscribed to that topic. SNS handles delivery, retries, and fan-out. Services consume independently at their own pace.

```
                      ┌──► SQS (Inventory) ──► Consumer B
                      │
Publisher ──► SNS ────┼──► SQS (Billing)   ──► Consumer C
                      │
                      └──► SQS (Analytics) ──► Consumer D
```

### Subscribing an SQS Queue to SNS (CLI)

```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:eu-west-1:123456789012:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:eu-west-1:123456789012:billing-queue \
  --attributes '{"RawMessageDelivery": "true"}'
```

### SQS Queue Policy (Allow SNS to Push)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "sns.amazonaws.com"},
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:eu-west-1:123456789012:billing-queue",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "arn:aws:sns:eu-west-1:123456789012:order-events"
        }
      }
    }
  ]
}
```

> **Gotcha:** Without `RawMessageDelivery: true`, SNS wraps your message in its own JSON envelope. Your consumer then has to unwrap `Message` from within an SNS notification object. Always enable raw delivery for SQS subscribers unless you need the SNS metadata.

---

## SNS Filter Policies

**Problem:** Your `order-events` topic publishes events for all order states: `created`, `paid`, `shipped`, `cancelled`. The billing queue only cares about `paid` — but without filtering, it receives (and discards) every message. At scale, this wastes compute and can cause consumer lag.

**Solution:** Attach a filter policy to the SNS subscription. SNS evaluates the policy against message attributes and only delivers matching messages to that subscriber.

### Filter Policy Example

```json
{
  "order_status": ["paid"],
  "region": [{"prefix": "eu-"}]
}
```

This policy delivers only messages where `order_status` is `paid` AND `region` starts with `eu-`.

### Publishing with Message Attributes

```bash
aws sns publish \
  --topic-arn arn:aws:sns:eu-west-1:123456789012:order-events \
  --message '{"orderId": "abc-123", "amount": 49.99}' \
  --message-attributes '{
    "order_status": {"DataType": "String", "StringValue": "paid"},
    "region": {"DataType": "String", "StringValue": "eu-west"}
  }'
```

### Applying the Filter Policy to a Subscription

```bash
aws sns set-subscription-attributes \
  --subscription-arn arn:aws:sns:eu-west-1:123456789012:order-events:a1b2c3d4 \
  --attribute-name FilterPolicy \
  --attribute-value '{"order_status": ["paid"], "region": [{"prefix": "eu-"}]}'
```

> **Pro-Tip:** By default, filter policies match against **message attributes**. Since 2022, you can set `FilterPolicyScope` to `MessageBody` to filter on the JSON payload itself — useful when you can't control the publisher's attributes. Set it like this:
>
> ```bash
> aws sns set-subscription-attributes \
>   --subscription-arn <sub-arn> \
>   --attribute-name FilterPolicyScope \
>   --attribute-value MessageBody
> ```

---

## Handling Out-of-Order Messages

**Problem:** Standard SQS queues do not guarantee message ordering. If you publish `order.updated (status=paid)` and then `order.updated (status=shipped)` in quick succession, the consumer might process `shipped` before `paid`. Your database now says "paid" when it should say "shipped."

**Approach: Timestamp-based idempotency at the consumer.**

Every message includes an `event_timestamp`. The consumer compares the incoming timestamp against the last-processed timestamp stored alongside the entity. If the incoming event is older, it's discarded.

### Message Schema

```json
{
  "event_type": "order.updated",
  "entity_id": "order-abc-123",
  "event_timestamp": "2026-03-06T14:30:00.123Z",
  "payload": {
    "status": "shipped",
    "tracking_number": "DHL-987654"
  }
}
```

### Consumer Pseudocode

```python
def handle_order_update(message):
    order = db.get_order(message["entity_id"])

    if order and order.last_event_at >= message["event_timestamp"]:
        log.info(f"Skipping stale event for {message['entity_id']}")
        return  # ACK the message, discard stale data

    db.upsert_order(
        entity_id=message["entity_id"],
        status=message["payload"]["status"],
        last_event_at=message["event_timestamp"]
    )
```

> **Gotcha:** Use the publisher's event timestamp, **not** the SQS `SentTimestamp` attribute. SQS timestamps reflect when the message entered the queue, not when the business event occurred. If a publisher retries a failed SNS publish, the SQS timestamp will be newer but the event is the same (or older).

> **Pro-Tip:** For strict ordering without consumer-side logic, use **SQS FIFO queues** with `MessageGroupId`. Messages within the same group are delivered in order. Trade-off: FIFO queues cap at 300 msg/s per group (3,000 with batching) vs. virtually unlimited for standard queues.

---

## Amazon ARN Structure

**Problem:** IAM policies, subscription configs, and cross-account access all reference resources by ARN. Misunderstanding the format leads to overly permissive policies or hard-to-debug "access denied" errors.

### ARN Format

```
arn:partition:service:region:account-id:resource-type/resource-id
```

### Breakdown

| Segment        | Example              | Notes                                     |
|----------------|----------------------|-------------------------------------------|
| `partition`    | `aws`                | `aws-cn` for China, `aws-us-gov` for GovCloud |
| `service`      | `sns`, `sqs`         | The AWS service namespace                 |
| `region`       | `eu-west-1`          | Omitted for global services (IAM, S3 bucket-level) |
| `account-id`   | `123456789012`       | 12-digit AWS account ID                   |
| `resource-type`| (varies)             | Some services use `:` separator, others `/` |
| `resource-id`  | `order-events`       | The resource name                         |

### Real Examples

```
# SNS Topic
arn:aws:sns:eu-west-1:123456789012:order-events

# SQS Queue
arn:aws:sqs:eu-west-1:123456789012:billing-queue

# Lambda Function
arn:aws:lambda:eu-west-1:123456789012:function:order-publisher

# IAM Role (no region — IAM is global)
arn:aws:iam::123456789012:role/lambda-exec-role
```

### Using ARNs in Subscription Policies

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::999888777666:root"},
    "Action": "sns:Subscribe",
    "Resource": "arn:aws:sns:eu-west-1:123456789012:order-events"
  }]
}
```

> **Gotcha:** SQS ARNs use `:` (colon) before the queue name, not `/`. It's `arn:aws:sqs:region:account:queue-name`, not `arn:aws:sqs:region:account/queue-name`. This trips people up because S3 and IAM use `/`.

---

## Lambda as Publisher to SNS

**Problem:** You have multiple event triggers (HTTP API, S3 upload, scheduled job) that should all publish to the same SNS topic. Maintaining separate codebases for each is wasteful when the core publish logic is identical.

**Pattern:** Deploy one codebase as multiple Lambda functions with different handlers. Each handler extracts event-specific data, transforms it to a canonical message format, and publishes to SNS. AWS routes the right event source to the right handler via the function's event trigger configuration.

### Project Structure

```
order-publishers/
├── handlers/
│   ├── api_handler.py       # Triggered by API Gateway
│   ├── s3_handler.py        # Triggered by S3 PutObject
│   └── scheduler_handler.py # Triggered by EventBridge rule
├── shared/
│   └── publisher.py         # Common SNS publish logic
└── template.yaml            # SAM / CloudFormation
```

### Shared Publisher Module

```python
import boto3
import json
import os
from datetime import datetime, timezone

sns = boto3.client("sns")
TOPIC_ARN = os.environ["ORDER_TOPIC_ARN"]

def publish_order_event(event_type: str, entity_id: str, payload: dict):
    sns.publish(
        TopicArn=TOPIC_ARN,
        Message=json.dumps({
            "event_type": event_type,
            "entity_id": entity_id,
            "event_timestamp": datetime.now(timezone.utc).isoformat(),
            "payload": payload
        }),
        MessageAttributes={
            "event_type": {
                "DataType": "String",
                "StringValue": event_type
            }
        }
    )
```

### Handler Example (API Gateway Trigger)

```python
from shared.publisher import publish_order_event

def handler(event, context):
    body = json.loads(event["body"])
    publish_order_event(
        event_type="order.created",
        entity_id=body["order_id"],
        payload=body
    )
    return {"statusCode": 202, "body": "accepted"}
```

### SAM Template (Abbreviated)

```yaml
Resources:
  ApiPublisher:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/api_handler.handler
      Runtime: python3.12
      Environment:
        Variables:
          ORDER_TOPIC_ARN: !Ref OrderTopic
      Policies:
        - SNSPublishMessagePolicy:
            TopicArn: !Ref OrderTopic
      Events:
        Api:
          Type: Api
          Properties:
            Path: /orders
            Method: post

  S3Publisher:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/s3_handler.handler
      Runtime: python3.12
      Environment:
        Variables:
          ORDER_TOPIC_ARN: !Ref OrderTopic
      Policies:
        - SNSPublishMessagePolicy:
            TopicArn: !Ref OrderTopic
      Events:
        S3Event:
          Type: S3
          Properties:
            Bucket: !Ref UploadBucket
            Events: s3:ObjectCreated:*

  OrderTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: order-events
```

> **Pro-Tip:** Each Lambda function gets its own IAM execution role by default (SAM generates it). Use `Policies` with SAM policy templates like `SNSPublishMessagePolicy` instead of crafting inline IAM — it's less error-prone and automatically scopes permissions to the specific topic ARN.

> **Gotcha:** Lambda's default timeout is 3 seconds. SNS publish calls typically complete in <100ms, but if your VPC-attached Lambda needs a NAT gateway to reach SNS, cold starts + network setup can blow past that. Either increase the timeout or use a **VPC endpoint for SNS** (`com.amazonaws.region.sns`) to keep traffic off the public internet and reduce latency.
