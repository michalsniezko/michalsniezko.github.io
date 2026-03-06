---
layout: default
title: "Lambda Code Sharing: Multi-Handler vs. Monolith Router"
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 5
---

## Lambda Code Sharing: Multi-Handler vs. Monolith Router

**Scenario:** You have 5 event sources (API Gateway, SQS, S3, EventBridge, SNS) that share 80% of their logic (validation, data access, SNS publishing). Do you deploy 5 separate Lambda functions from the same codebase, or one Lambda with an internal router?

### Option A: Multiple Functions, Different Handlers (Recommended)

```
shared-codebase/
├── handlers/
│   ├── api.js            # API Gateway → handler
│   ├── sqsConsumer.js    # SQS → handler
│   ├── s3Trigger.js      # S3 → handler
│   └── scheduler.js      # EventBridge → handler
├── core/
│   ├── service.js        # Shared business logic
│   └── publisher.js      # Shared SNS publish
└── template.yaml         # SAM: 4 functions, 1 codebase
```

```yaml
# template.yaml
Resources:
  ApiHandler:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/api.handler
      Runtime: nodejs20.x
      MemorySize: 256
      Timeout: 10
      Events:
        Api:
          Type: Api
          Properties: { Path: /orders, Method: post }

  SqsConsumer:
    Type: AWS::Serverless::Function
    Properties:
      Handler: handlers/sqsConsumer.handler
      Runtime: nodejs20.x
      MemorySize: 512        # needs more memory for batch processing
      Timeout: 60            # SQS batches take longer
      ReservedConcurrentExecutions: 30
      Events:
        Queue:
          Type: SQS
          Properties:
            Queue: !GetAtt OrderQueue.Arn
            BatchSize: 10
```

### Option B: Monolith Router (Anti-Pattern at Scale)

```javascript
// handler.js - single entry point for everything
exports.handler = async (event, context) => {
    if (event.httpMethod) {
        return handleApi(event);
    } else if (event.Records?.[0]?.eventSource === 'aws:sqs') {
        return handleSqs(event);
    } else if (event.Records?.[0]?.eventSource === 'aws:s3') {
        return handleS3(event);
    } else {
        throw new Error(`Unknown event source: ${JSON.stringify(event).slice(0, 200)}`);
    }
};
```

### Comparison

| Aspect                    | Multi-Handler                      | Monolith Router                    |
|---------------------------|------------------------------------|------------------------------------|
| Memory/Timeout tuning     | Per function - optimized           | One size fits all - wasteful       |
| Concurrency limits        | Per function - isolated            | Shared - SQS spike starves API    |
| Error blast radius        | Bug in S3 handler doesn't affect API | Bug in router breaks everything |
| Cold starts               | Each function has its own cold start pool | Single pool, fewer cold starts |
| Deployment                | Can deploy one function independently | All-or-nothing deploy            |
| Monitoring                | Metrics per function - clear       | One set of metrics - noisy        |

> **Cost/Performance Note:** The monolith router's only real advantage is fewer cold starts (one warm pool serves all events). But this is offset by the inability to right-size memory and timeout per event type. An API handler needs 256MB and 10s; an SQS batch processor needs 512MB and 60s. With the monolith, you set 512MB/60s for everything - overpaying for API calls and under-provisioning nothing. At scale, the wasted memory across millions of API invocations far exceeds the cold start savings.
