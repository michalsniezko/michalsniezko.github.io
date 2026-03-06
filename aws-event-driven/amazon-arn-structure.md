---
layout: default
title: Amazon ARN Structure
parent: AWS Event-Driven Architecture
nav_order: 4
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

# IAM Role (no region - IAM is global)
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
