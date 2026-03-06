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

Adding a new consumer (e.g., a fraud detection service) requires zero changes to the publisher - just add a new SQS queue and SNS subscription with the appropriate filter.

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

> **Cost/Performance Note:** SNS charges per publish ($0.50/million requests), not per subscription delivery. But each SQS queue that receives the message incurs SQS costs. An unfiltered subscription on a high-volume topic (10M messages/day) where the consumer discards 95% of messages wastes ~$3.80/day in SQS receive + delete API calls alone. Always apply the narrowest filter policy possible - you're paying for every message that lands in a queue, even if the consumer immediately discards it.
