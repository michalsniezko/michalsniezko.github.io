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

```python
import boto3

# Initialized ONCE per Lambda container — persists across warm invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('orders')
sns = boto3.client('sns')

def handler(event, context):
    for record in event['Records']:
        body = json.loads(record['body'])
        # Reuses the existing connection pool
        table.put_item(Item=body)
        sns.publish(TopicArn=TOPIC_ARN, Message=record['body'])
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

> **Cost/Performance Note:** `reserved_concurrent_executions` protects downstream services (a database that can handle 50 connections, not 500). But it also means excess invocations get **throttled** — messages return to the SQS queue and retry. Set `reserved_concurrent_executions` based on what your downstream dependencies can handle, not what Lambda can scale to. Monitor the `Throttles` metric to confirm you're not silently dropping work.
