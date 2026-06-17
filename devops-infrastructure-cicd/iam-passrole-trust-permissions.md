---
layout: default
title: IAM PassRole, Trust Policies, and Permissions Policies
parent: Infrastructure & CI/CD
nav_order: 12
---

## IAM PassRole, Trust Policies, and Permissions Policies

IAM has three distinct concepts that are easy to confuse because they all affect what a role can do - but they control completely different things. Getting any one of them wrong results in an `AccessDenied` error with no obvious explanation.

---

## The Three Concepts

**Trust policy** - who is allowed to assume this role. It answers: "which principal can call `sts:AssumeRole` and become this role?" Every role has exactly one trust policy.

**Permissions policy** - what the role is allowed to do once assumed. It answers: "which AWS API calls can this role make?" A role can have multiple permissions policies attached.

**`iam:PassRole`** - a permission that controls which roles a principal is allowed to hand off to an AWS service. It answers: "is this caller allowed to tell AWS Service X to use Role Y?"

They work in combination:

```
You (caller)                 AWS Service              Target Role
    |                             |                        |
    |-- iam:PassRole on Role Y -->|                        |
    |-- "use Role Y" ------------>|                        |
                                  |-- sts:AssumeRole ----->|
                                  |     (trust policy)     |
                                  |<-- credentials --------|
                                  |-- API calls ---------->| (permissions policy)
```

---

## Trust Policy

The trust policy is attached to the role itself and defines who can assume it. The principal can be an AWS service, another role, a user, or an external identity provider.

### Service trust policy

Most service roles have a trust policy that allows the AWS service to assume the role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

This says: "ECS task containers are allowed to assume this role." Without this, ECS cannot call `sts:AssumeRole` on the role and the task has no credentials.

### Role-to-role trust (cross-account or same-account)

A role in account A can trust a role in account B:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/ci-pipeline-role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Trust policy with conditions

Conditions narrow who can assume the role even within the trusted principal:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "aws:SourceAccount": "123456789012"
        }
      }
    }
  ]
}
```

For OIDC-based federation (GitHub Actions, Kubernetes service accounts), conditions are essential - without them, any repository or pod using the same OIDC provider could assume the role:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:your-org/your-repo:*"
    }
  }
}
```

---

## Permissions Policy

The permissions policy (or multiple policies attached to the role) defines what the assumed role can do. This is the standard IAM policy format:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:eu-west-1:123456789012:my-queue"
    }
  ]
}
```

A role's effective permissions are the intersection of:
- What the trust policy allows (who can assume it)
- What the permissions policies allow (what it can do once assumed)
- What permissions boundaries allow (if set - an upper limit on effective permissions)

---

## iam:PassRole

`iam:PassRole` is a special permission that prevents privilege escalation. Without it, any caller who can create or update an AWS resource with an associated role could grant that resource more permissions than they themselves have.

The scenario it prevents:

```
Attacker has: permission to create Lambda functions
             no permission to access S3

Without iam:PassRole check:
  Attacker creates Lambda with an admin role attached
  Lambda can now access S3 (via the admin role)
  Attacker has effectively escalated privileges

With iam:PassRole check:
  AWS checks: does the attacker have iam:PassRole on the admin role?
  No -> AccessDenied when creating the Lambda
```

Every time you configure a service to use a role, `iam:PassRole` is checked on the caller - not the service, not the role. The caller creating or updating the resource must have it.

### Common situations that require iam:PassRole

- Creating an ECS task definition with `taskRoleArn` or `executionRoleArn`
- Updating an ECS service with a new task definition
- Creating a Lambda with an execution role
- Creating a CloudWatch Events rule target that triggers ECS with a role
- Creating an EC2 instance with an instance profile
- Creating a CodePipeline with service roles

### iam:PassRole policy

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/my-ecs-task-role",
  "Condition": {
    "StringEquals": {
      "iam:PassedToService": "ecs-tasks.amazonaws.com"
    }
  }
}
```

The `iam:PassedToService` condition scopes the permission: this caller can pass the role, but only to ECS - not to Lambda, EC2, or any other service.

Without the condition, the permission is:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/my-ecs-task-role"
}
```

Still valid, just less restrictive.

---

## How They Work Together: An ECS Example

A CloudWatch Events rule triggers an ECS task on a schedule. Three principals are involved, each needing the right policies:

**1. The CloudWatch Events service needs to start ECS tasks.**

It assumes a trigger role. That role's trust policy must allow `events.amazonaws.com`:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "events.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Its permissions policy must allow running the task:

```json
{
  "Effect": "Allow",
  "Action": ["ecs:RunTask"],
  "Resource": "arn:aws:ecs:eu-west-1:123456789012:task-definition/my-task:*"
}
```

**But this alone is not enough.** When CloudWatch calls `ecs:RunTask` it also passes the task's execution role and task role to ECS. That requires `iam:PassRole` on those roles:

```json
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": [
    "arn:aws:iam::123456789012:role/ecs-task-execution-role",
    "arn:aws:iam::123456789012:role/ecs-task-role"
  ],
  "Condition": {
    "StringLike": {
      "iam:PassedToService": "ecs-tasks.amazonaws.com"
    }
  }
}
```

**2. The ECS task execution role** needs to pull the Docker image from ECR and fetch secrets from SSM. Its trust policy allows ECS:

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "ecs-tasks.amazonaws.com" },
  "Action": "sts:AssumeRole"
}
```

Its permissions policy allows ECR and SSM access:

```json
{
  "Effect": "Allow",
  "Action": [
    "ecr:GetAuthorizationToken",
    "ecr:BatchGetImage",
    "ssm:GetParameters",
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "*"
}
```

**3. The ECS task role** is what the running container uses for its own API calls (S3, SQS, etc.). Same trust policy as the execution role, different permissions policy based on what the application needs.

---

## Common Pitfalls

### Pitfall 1: Forgetting iam:PassRole when updating a service

You have `ecs:UpdateService` permission. You update a service to use a new task definition. The new task definition uses a different task role.

Result: `AccessDenied` - not on `ecs:UpdateService` but on `iam:PassRole` for the new task role.

The fix: ensure the caller has `iam:PassRole` on every role referenced in the task definition, not just the service-level role.

---

### Pitfall 2: Wrong service principal in the trust policy

Trust policies use the service's principal identifier, which is not always obvious:

```json
// Wrong - this service does not exist
{ "Service": "ecs.amazonaws.com" }

// Correct for ECS task roles
{ "Service": "ecs-tasks.amazonaws.com" }

// Correct for EC2 instance profiles
{ "Service": "ec2.amazonaws.com" }

// Correct for Lambda
{ "Service": "lambda.amazonaws.com" }

// Correct for CloudWatch Events / EventBridge
{ "Service": "events.amazonaws.com" }

// Correct for CodeBuild
{ "Service": "codebuild.amazonaws.com" }
```

A wrong service principal causes a silent failure: the trust policy is valid JSON and Terraform applies it successfully, but the service cannot assume the role at runtime.

---

### Pitfall 3: Confusing the trust policy with the permissions policy

A common mistake is adding `sts:AssumeRole` to the permissions policy instead of the trust policy:

```json
// Wrong: this goes in the PERMISSIONS policy of the CALLER,
// not in the trust policy of the role being assumed
{
  "Effect": "Allow",
  "Action": "sts:AssumeRole",
  "Resource": "arn:aws:iam::123456789012:role/my-role"
}
```

```json
// Correct: this goes in the TRUST POLICY of the role being assumed
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::123456789012:role/caller-role" },
  "Action": "sts:AssumeRole"
}
```

Both are needed for cross-account role assumption:
- The caller's permissions policy must allow `sts:AssumeRole` on the target role ARN
- The target role's trust policy must allow the caller principal

For same-account assumption, the trust policy alone is sufficient - the `sts:AssumeRole` permission in the caller's policy is only required cross-account.

---

### Pitfall 4: Overly broad iam:PassRole

Using `Resource: "*"` on `iam:PassRole` means the caller can hand any role in the account to any service. This is a privilege escalation risk: the caller could pass an admin role to a Lambda they control.

Always scope `iam:PassRole` to the specific role ARNs needed, and use `iam:PassedToService` to lock it to the specific service:

```json
// Risky
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "*"
}

// Better
{
  "Effect": "Allow",
  "Action": "iam:PassRole",
  "Resource": "arn:aws:iam::123456789012:role/ecs-task-role-*",
  "Condition": {
    "StringEquals": {
      "iam:PassedToService": "ecs-tasks.amazonaws.com"
    }
  }
}
```

---

### Pitfall 5: iam:PassRole is not transitive

If Service A assumes Role X, and then calls a service that requires passing Role Y, `iam:PassRole` must be in Role X's permissions policy - not in the caller that originally assumed Role X.

Example: a CI pipeline (Role X) creates an ECS task definition. Role X must have `iam:PassRole` on the task execution role. The human or CI service that assumed Role X does not need `iam:PassRole` for this operation - the check is against Role X, not against the original caller.

---

### Pitfall 6: Trust policy allows a role that no longer exists

If you update a trust policy to trust `arn:aws:iam::123456789012:role/old-pipeline-role` and that role is later deleted and recreated (even with the same name), the trust relationship breaks. IAM internally tracks roles by a unique ID (`AROA...`), not just the ARN. Deleting and recreating a role generates a new unique ID.

The fix: always re-apply the trust policy after recreating a trusted role.

---

### Pitfall 7: The execution role and the task role are different things

In ECS, there are two roles on a task definition:

**Execution role** (`executionRoleArn`) - used by the ECS agent to pull the image and fetch secrets before the container starts. The application code never uses this role.

**Task role** (`taskRoleArn`) - used by the application running inside the container for its own API calls (S3, SQS, DynamoDB, etc.).

Giving the task role permission to access ECR (because the execution role needs it) has no effect - the task role is not used for image pulls. And giving the execution role permission to access SQS has no effect on the application - the container uses the task role for that.

---

## Terraform Reference

A complete ECS setup with all three policies correct:

```hcl
# The execution role - used by ECS agent, not the application
resource "aws_iam_role" "ecs_execution" {
  name = "ecs-execution-${var.service_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution_managed" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# The task role - used by the application inside the container
resource "aws_iam_role" "ecs_task" {
  name = "ecs-task-${var.service_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "ecs_task_permissions" {
  name = "app-permissions"
  role = aws_iam_role.ecs_task.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["sqs:ReceiveMessage", "sqs:DeleteMessage"]
      Resource = var.queue_arn
    }]
  })
}

# The CloudWatch trigger role - used by EventBridge to start ECS tasks
resource "aws_iam_role" "cloudwatch_trigger" {
  name = "cloudwatch-ecs-trigger-${var.service_name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "events.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "cloudwatch_trigger_permissions" {
  name = "run-ecs-task"
  role = aws_iam_role.cloudwatch_trigger.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ecs:RunTask"]
        Resource = "${aws_ecs_task_definition.task.arn_without_revision}:*"
      },
      {
        Effect = "Allow"
        Action = "iam:PassRole"
        Resource = [
          aws_iam_role.ecs_execution.arn,
          aws_iam_role.ecs_task.arn,
        ]
        Condition = {
          StringLike = {
            "iam:PassedToService" = "ecs-tasks.amazonaws.com"
          }
        }
      }
    ]
  })
}
```

---

### For AI agents

```
IAM has three distinct concepts: trust policy (who can assume the role - sts:AssumeRole in the role's own policy document), permissions policy (what the assumed role can do), and iam:PassRole (whether a caller can hand a role to an AWS service). Common pitfalls: forgetting iam:PassRole when creating/updating resources with associated roles; wrong service principal in trust policy (ecs-tasks.amazonaws.com not ecs.amazonaws.com); confusing trust policy with permissions policy; iam:PassRole with Resource:* is a privilege escalation risk - always scope to specific role ARNs and use iam:PassedToService condition; iam:PassRole is not transitive - check it on the role making the API call, not the original human caller. ECS has two separate roles: executionRoleArn (ECS agent pulls image and secrets) and taskRoleArn (application code makes API calls) - they need different permissions.
```

Reference: `https://michalsniezko.github.io/devops-infrastructure-cicd/iam-passrole-trust-permissions.html`
