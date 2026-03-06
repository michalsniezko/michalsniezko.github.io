---
layout: default
title: SNS Filter Policies
parent: AWS Event-Driven Architecture
nav_order: 2
---

## SNS Filter Policies

**Problem:** Your `order-events` topic publishes events for all order states: `created`, `paid`, `shipped`, `cancelled`. The billing queue only cares about `paid` - but without filtering, it receives (and discards) every message. At scale, this wastes compute and can cause consumer lag.

**Solution:** Attach a filter policy to the SNS subscription. SNS evaluates the policy against message attributes and only delivers matching messages to that subscriber.

### Filter Policy Example

```json
{
  "order_status": ["paid"],
  "region": [{"prefix": "eu-"}]
}
```

This policy delivers only messages where `order_status` is `paid` AND `region` starts with `eu-`.

### Publishing with Message Attributes (PHP)

```php
use Aws\Sns\SnsClient;

$sns = new SnsClient(['region' => 'eu-west-1', 'version' => 'latest']);

$sns->publish([
    'TopicArn' => 'arn:aws:sns:eu-west-1:123456789012:order-events',
    'Message'  => json_encode(['orderId' => 'abc-123', 'amount' => 49.99]),
    'MessageAttributes' => [
        'order_status' => ['DataType' => 'String', 'StringValue' => 'paid'],
        'region'       => ['DataType' => 'String', 'StringValue' => 'eu-west'],
    ],
]);
```

### Terraform: Subscription with Filter on Message Attributes

```hcl
resource "aws_sns_topic_subscription" "billing_paid_orders" {
  topic_arn = aws_sns_topic.order_events.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.billing.arn

  filter_policy = jsonencode({
    order_status = ["paid"]
    region       = [{ prefix = "eu-" }]
  })
}
```

### Terraform: Filter on Message Body

By default, `filter_policy` matches against **message attributes**. Set `filter_policy_scope` to `MessageBody` to filter on fields inside the JSON payload itself - useful when you can't control the publisher's attributes.

```hcl
resource "aws_sns_topic_subscription" "car_changelog" {
  topic_arn           = var.car_changelog_sns_topic_arn
  protocol            = "sqs"
  endpoint            = aws_sqs_queue.car_changelog.arn
  filter_policy_scope = "MessageBody"
  filter_policy       = jsonencode({ type = ["Entity.Car.update"] })
}
```

Both `filter_policy_scope` and `filter_policy` live on the same subscription resource - SNS uses the scope to decide whether to evaluate the filter against message attributes or the message body.

> **Gotcha:** If you set `filter_policy_scope = "MessageBody"` but your publisher sends the routing info as message attributes (not in the JSON body), the filter will never match and the subscription silently receives nothing. Make sure the scope matches where your publisher actually puts the data.
