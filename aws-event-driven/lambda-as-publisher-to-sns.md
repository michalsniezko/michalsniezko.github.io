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
│   ├── apiHandler.js        # Triggered by API Gateway
│   ├── s3Handler.js         # Triggered by S3 PutObject
│   └── schedulerHandler.js  # Triggered by EventBridge rule
├── shared/
│   └── publisher.js         # Common SNS publish logic
└── template.yaml            # SAM / CloudFormation
```

### Shared Publisher Module

```javascript
// shared/publisher.js
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

const sns = new SNSClient({});
const TOPIC_ARN = process.env.ORDER_TOPIC_ARN;

async function publishOrderEvent(eventType, entityId, payload) {
    await sns.send(new PublishCommand({
        TopicArn: TOPIC_ARN,
        Message: JSON.stringify({
            event_type: eventType,
            entity_id: entityId,
            event_timestamp: new Date().toISOString(),
            payload,
        }),
        MessageAttributes: {
            event_type: {
                DataType: 'String',
                StringValue: eventType,
            },
        },
    }));
}

module.exports = { publishOrderEvent };
```

### Handler Example (API Gateway Trigger)

```javascript
// handlers/apiHandler.js
const { publishOrderEvent } = require('../shared/publisher');

exports.handler = async (event) => {
    const body = JSON.parse(event.body);
    await publishOrderEvent('order.created', body.order_id, body);
    return { statusCode: 202, body: 'accepted' };
};
```

### SAM Template (Abbreviated)

```yaml
Resources:
  ApiPublisher:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/apiHandler.handler
      Runtime: nodejs20.x
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
      Handler: handlers/s3Handler.handler
      Runtime: nodejs20.x
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

> **Pro-Tip:** Each Lambda function gets its own IAM execution role by default (SAM generates it). Use `Policies` with SAM policy templates like `SNSPublishMessagePolicy` instead of crafting inline IAM - it's less error-prone and automatically scopes permissions to the specific topic ARN.

> **Gotcha:** Lambda's default timeout is 3 seconds. SNS publish calls typically complete in <100ms, but if your VPC-attached Lambda needs a NAT gateway to reach SNS, cold starts + network setup can blow past that. Either increase the timeout or use a **VPC endpoint for SNS** (`com.amazonaws.region.sns`) to keep traffic off the public internet and reduce latency.
