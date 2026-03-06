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

> **Pro-Tip:** By default, filter policies match against **message attributes**. Since 2022, you can set `FilterPolicyScope` to `MessageBody` to filter on the JSON payload itself - useful when you can't control the publisher's attributes. Set it like this:
>
> ```bash
> aws sns set-subscription-attributes \
>   --subscription-arn <sub-arn> \
>   --attribute-name FilterPolicyScope \
>   --attribute-value MessageBody
> ```
