---
layout: default
title: AWS SSM Parameter Store
parent: Infrastructure as Code
nav_order: 4
---

## AWS SSM Parameter Store

**Flow:** Engineer stores secret in SSM → application reads it at boot (or Terraform references it during deploy) → no secrets in Git, no secrets in environment variable definitions committed to repos.

Hardcoding `DB_PASSWORD=hunter2` in a `.env` file that gets committed (even to a private repo) is a matter of time before it leaks. [SSM Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) provides encrypted, versioned, auditable secret storage.

### Storing Parameters (Terraform)

```hcl
# String parameter (non-sensitive config)
resource "aws_ssm_parameter" "db_host" {
  name  = "/order-service/prod/database/host"
  type  = "String"
  value = "orders-db.cluster-abc123.eu-west-1.rds.amazonaws.com"
}

# SecureString (encrypted with KMS)
resource "aws_ssm_parameter" "db_password" {
  name   = "/order-service/prod/database/password"
  type   = "SecureString"
  value  = var.db_password  # passed via CI variable, never hardcoded
  key_id = "alias/order-service-key"
}
```

### Referencing in Terraform (ECS Task Definition)

```hcl
resource "aws_ecs_task_definition" "order_service" {
  family                = "order-service"
  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${var.ecr_repo}:${var.image_tag}"
      secrets = [
        {
          name      = "DB_HOST"
          valueFrom = "arn:aws:ssm:eu-west-1:123456789012:parameter/order-service/prod/database/host"
        },
        {
          name      = "DB_PASSWORD"
          valueFrom = "arn:aws:ssm:eu-west-1:123456789012:parameter/order-service/prod/database/password"
        }
      ]
    }
  ])

  execution_role_arn = aws_iam_role.ecs_execution.arn
}
```

### IAM Policy for ECS to Read Secrets

```hcl
resource "aws_iam_role_policy" "ecs_ssm_access" {
  name = "ecs-ssm-read"
  role = aws_iam_role.ecs_execution.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ssm:GetParameters", "ssm:GetParameter"]
        Resource = "arn:aws:ssm:eu-west-1:123456789012:parameter/order-service/prod/*"
      },
      {
        Effect   = "Allow"
        Action   = ["kms:Decrypt"]
        Resource = "arn:aws:kms:eu-west-1:123456789012:key/key-id-here"
      }
    ]
  })
}
```

### Reading in PHP at Runtime (Fallback for Non-ECS)

```php
use Aws\Ssm\SsmClient;

class ParameterStoreConfig
{
    private SsmClient $ssm;

    public function __construct(string $region = 'eu-west-1')
    {
        $this->ssm = new SsmClient(['region' => $region, 'version' => 'latest']);
    }

    public function get(string $path): string
    {
        $result = $this->ssm->getParameter([
            'Name'           => $path,
            'WithDecryption' => true,
        ]);

        return $result['Parameter']['Value'];
    }

    /** @return array<string, string> */
    public function getByPath(string $prefix): array
    {
        $result = $this->ssm->getParametersByPath([
            'Path'           => $prefix,
            'WithDecryption' => true,
            'Recursive'      => true,
        ]);

        $params = [];
        foreach ($result['Parameters'] as $param) {
            $key = basename($param['Name']);
            $params[$key] = $param['Value'];
        }

        return $params;
    }
}
```

> **Security Note:** SSM `GetParameter` calls are logged in [CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html) - you get a full audit trail of who accessed which secret and when. But: if your application logs the values it reads (`$logger->info('DB config', $params)`), you've just written the plaintext secret to your log aggregator. Treat SSM values as opaque - log the parameter *name*, never the *value*.
