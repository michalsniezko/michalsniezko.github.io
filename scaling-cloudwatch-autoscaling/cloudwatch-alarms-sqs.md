---
layout: default
title: CloudWatch Alarms for SQS Backlog
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 2
---

## CloudWatch Alarms for SQS Backlog

**Metric:** `ApproximateNumberOfMessagesVisible` (for backlog) and [`ApproximateAgeOfOldestMessage`](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-available-cloudwatch-metrics.html) (for staleness). Age is often more actionable - 1,000 messages that are 2 seconds old is fine; 10 messages that are 30 minutes old means your consumer is dead.

### Terraform: Queue Depth Alarm

[`aws_cloudwatch_metric_alarm`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_metric_alarm) triggers SNS notifications when thresholds are breached:

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
  alarm_description   = "Oldest message in queue is > 10 minutes old - consumer may be stuck"

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

> **Cost/Performance Note:** `evaluation_periods = 3` with `period = 60` means the alarm triggers after 3 consecutive minutes above threshold. This prevents alert noise from brief spikes (e.g., a batch job drops 10k messages, workers drain it in 90 seconds). For `ApproximateAgeOfOldestMessage`, use `statistic = "Maximum"` - you care about the worst case, not the average.
