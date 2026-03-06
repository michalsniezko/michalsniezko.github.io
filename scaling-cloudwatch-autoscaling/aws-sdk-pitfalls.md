---
layout: default
title: AWS SDK Pitfalls in High-Concurrency Environments
parent: CloudWatch, Autoscaling, Lambda Patterns
nav_order: 4
---

## AWS SDK Pitfalls in High-Concurrency Environments

**Scenario:** Your Lambda processes SQS batches of 10 records. Each record triggers 3 AWS API calls (DynamoDB get, S3 put, SNS publish). At 100 concurrent Lambda invocations, that's 3,000 near-simultaneous API calls. The SDK's default connection handling wasn't designed for this.

### Common Issues

| Problem                       | Symptom                                        | Fix                                         |
|-------------------------------|------------------------------------------------|---------------------------------------------|
| Connection exhaustion         | `TooManyRequestsException` or socket timeouts  | Reuse SDK client outside the handler        |
| Cold start latency            | First invocation takes 3–5 seconds             | Initialize clients at module level          |
| Memory leak (PHP/Node)        | Lambda memory grows until OOM kill             | Don't create new clients per invocation     |
| Retry storm                   | Cascading failures under load                  | Configure max retries + exponential backoff |

### Client Reuse (Module-Level Init)

The [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/) uses a modular client-per-service design — initialize clients once at module level, not per invocation:

```javascript
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient, PutCommand } = require('@aws-sdk/lib-dynamodb');
const { SNSClient, PublishCommand } = require('@aws-sdk/client-sns');

// Initialized ONCE per Lambda container - persists across warm invocations
const dynamodb = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const sns = new SNSClient({});

exports.handler = async (event) => {
    for (const record of event.Records) {
        const body = JSON.parse(record.body);
        // Reuses the existing connection pool
        await dynamodb.send(new PutCommand({ TableName: 'orders', Item: body }));
        await sns.send(new PublishCommand({ TopicArn: TOPIC_ARN, Message: record.body }));
    }
};
```

### PHP SDK: Retry Configuration

```php
use Aws\Sdk;

$sdk = new Sdk([
    'region'  => 'eu-west-1',
    'version' => 'latest',
    'retries' => [
        'mode'        => 'adaptive',  // token-bucket based backoff
        'max_attempts' => 3,
    ],
    'http' => [
        'connect_timeout' => 2,
        'timeout'         => 5,
    ],
]);

// Create clients from the shared SDK instance
$sqs = $sdk->createSqs();
$sns = $sdk->createSns();
```

### Lambda Concurrency Guardrails

```hcl
resource "aws_lambda_function" "order_processor" {
  function_name = "order-processor"
  # ...

  reserved_concurrent_executions = 50  # cap at 50 simultaneous invocations
}
```

> **Cost/Performance Note:** `reserved_concurrent_executions` protects downstream services (a database that can handle 50 connections, not 500). But it also means excess invocations get **throttled** - messages return to the SQS queue and retry. Set `reserved_concurrent_executions` based on what your downstream dependencies can handle, not what Lambda can scale to. Monitor the `Throttles` metric to confirm you're not silently dropping work.
