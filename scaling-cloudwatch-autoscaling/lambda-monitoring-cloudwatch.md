---
layout: default
title: Lambda Monitoring with CloudWatch
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 3
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

### Structured Logging for [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)

```javascript
exports.handler = async (event, context) => {
    const start = Date.now();

    // ... business logic ...

    const durationMs = Date.now() - start;
    const remainingMs = context.getRemainingTimeInMillis();

    console.log(JSON.stringify({
        event: 'invocation_complete',
        function: context.functionName,
        request_id: context.awsRequestId,
        duration_ms: durationMs,
        remaining_ms: remainingMs,
        memory_limit_mb: parseInt(context.memoryLimitInMB, 10),
        records_processed: (event.Records || []).length,
    }));
};
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

> **Cost/Performance Note:** CloudWatch Logs charges per GB ingested. A Lambda processing 1M events/day with verbose logging can generate 10+ GB/month of logs (~$5/GB). Log at `INFO` in production, never `DEBUG`. Use structured JSON logging so you can query with CloudWatch Insights instead of dumping everything and grepping later. Set a log retention policy (14–30 days) - unset retention means logs live forever and costs grow silently.
