---
layout: default
title: Lambda as Publisher to SNS
parent: AWS Event-Driven Architecture
nav_order: 5
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
