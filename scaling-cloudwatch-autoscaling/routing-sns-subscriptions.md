---
layout: default
title: Routing via SNS Subscriptions
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 6
---

## Routing via SNS Subscriptions

**Metric:** No custom metric needed - routing is declarative via subscription configuration. Monitor `NumberOfMessagesPublished` (SNS) and `NumberOfMessagesReceived` (SQS) to verify messages flow.

**Scenario:** Your order service publishes `order.created`, `order.paid`, and `order.cancelled` events. The billing service only needs `order.paid`. The analytics service needs everything. The notification service needs `order.paid` and `order.cancelled`. Implementing this routing in application code means every publisher needs to know about every consumer.

**Solution:** Publish everything to one SNS topic. Routing lives in subscription filter policies - the publisher is completely unaware of who consumes what.

### Architecture

```
                                 Filter: order_status=["paid"]
Order Service ──► SNS Topic ──┬──────────────────────────────► SQS (Billing)
                              │
                              │  Filter: (none - receives all)
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

```javascript
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

const sns = new SNSClient({});

await sns.send(new PublishCommand({
    TopicArn: TOPIC_ARN,
    Message: JSON.stringify({ orderId: 'ord-456', amount: 199.99 }),
    MessageAttributes: {
        order_status: { DataType: 'String', StringValue: 'paid' },
        region:       { DataType: 'String', StringValue: 'eu-west' },
    },
}));
```

Adding a new consumer (e.g., a fraud detection service) requires zero changes to the publisher - just add a new SQS queue and SNS subscription with the appropriate filter.

### Verifying Routing (Terraform Output)

```hcl
output "billing_subscription_arn" {
  value = aws_sns_topic_subscription.billing.arn
}

output "billing_filter_policy" {
  value = aws_sns_topic_subscription.billing.filter_policy
}

output "analytics_subscription_arn" {
  value = aws_sns_topic_subscription.analytics.arn
}

output "notifications_filter_policy" {
  value = aws_sns_topic_subscription.notifications.filter_policy
}
```

Run `terraform output` after apply to confirm each subscription has the expected filter policy. For runtime verification, check the SNS delivery metrics in CloudWatch: `NumberOfMessagesPublished` on the topic vs. `NumberOfMessagesReceived` on each queue.

> **Cost/Performance Note:** SNS charges per publish ($0.50/million requests), not per subscription delivery. But each SQS queue that receives the message incurs SQS costs. An unfiltered subscription on a high-volume topic (10M messages/day) where the consumer discards 95% of messages wastes ~$3.80/day in SQS receive + delete API calls alone. Always apply the narrowest filter policy possible - you're paying for every message that lands in a queue, even if the consumer immediately discards it.
