---
layout: default
title: ECS Standalone and Scheduled Tasks
parent: Infrastructure & CI/CD
nav_order: 11
---

## ECS Standalone and Scheduled Tasks

Not every piece of work is an HTTP request. Database migrations, backfills, periodic syncs, and one-off maintenance commands all need to run in the same environment as the service - with the same secrets, network access, and Docker image - but outside the normal request lifecycle. ECS handles this with two patterns: standalone tasks (triggered manually via Jenkins) and scheduled tasks (triggered automatically by CloudWatch Events on a cron schedule).

---

## Why Not Run This in the Application Entrypoint

The tempting shortcut is to run migrations or one-off jobs at startup. This fails for several reasons:

- All ECS tasks start simultaneously, so the migration runs multiple times in parallel
- ECS kills containers that fail health checks too slowly - a long migration gets interrupted mid-way
- MySQL does not have transactions for DDL statements. An interrupted migration can leave the schema in an inconsistent state

Running migrations in the entrypoint is the worst option. A dedicated task is the right approach.

---

## Standalone Tasks: One-Off Commands via Jenkins

A standalone task uses the same Docker image as the service but runs a specific command and exits. It is triggered manually from Jenkins, not from the main deployment pipeline.

### Directory structure

The standalone task has its own Terraform directory, separate from the main service infrastructure:

```
infrastructure/
  jenkins/
    standalone.groovy      # Jenkins job definition
  terraform/
    one-time-task/
      main.tf              # registers the ECS task definition
      task.json            # container definition template
```

### task.json: the container definition

The container runs a command and exits. The `%COMMAND%` placeholder is filled in by the Jenkins pipeline at runtime:

```json
[
  {
    "name": "app",
    "image": "${registry}/${image_name}:${version}",
    "command": %COMMAND%,
    "cpu": ${cpu},
    "memory": ${memory},
    "essential": true,
    "logConfiguration": {
        "logDriver": "json-file",
        "options": ${log_options}
    },
    "environment": ${environment_variables},
    "secrets": ${secrets}
  }
]
```

No health check - the task exits with code 0 on success or non-zero on failure. Jenkins polls for the exit code.

### main.tf: registering the task definition

```hcl
variable "service_name" {}
variable "image_name"   {}
variable "app_version"  {}
variable "registry"     {}
variable "cpu"          {}
variable "memory"       {}
variable "team"         {}

variable "task_type" {
  type    = string
  default = "one-time-task"
}

variable "environment_variables" {
  type = list(object({ name = string, value = string }))
}

variable "secrets" {
  type = list(object({ name = string, valueFrom = string }))
}

variable "dynamic_envs" {
  type = list(object({ name = string, value = string }))
}

locals {
  is_prod     = replace(terraform.workspace, "prod", "") != terraform.workspace
  environment = local.is_prod ? "prod" : "qa"

  iam_role_arn = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:role/ecs-task-${var.service_name}-${terraform.workspace}"
  task_family  = "${var.service_name}-${var.task_type}-${terraform.workspace}"
}

data "aws_caller_identity" "current" {}

resource "aws_ecs_task_definition" "task" {
  container_definitions = templatefile("task.json", {
    image_name            = var.image_name
    registry              = var.registry
    version               = var.app_version
    cpu                   = var.cpu
    memory                = var.memory
    environment_variables = jsonencode(concat(var.environment_variables, var.dynamic_envs))
    secrets               = jsonencode(var.secrets)
    log_options           = jsonencode({ "tag" = "docker/${var.service_name}" })
  })

  family             = local.task_family
  task_role_arn      = local.iam_role_arn
  execution_role_arn = local.iam_role_arn
  network_mode       = "bridge"

  tags = {
    "service" = var.service_name
    "env"     = terraform.workspace
    "team"    = var.team
  }
}
```

The `task_family` naming convention is `{service_name}-one-time-task-{workspace}`. ECS uses this to version the task definition - each Terraform apply registers a new revision.

### standalone.groovy: the Jenkins pipeline

The standalone pipeline does four things: register the task definition via Terraform, start the task with `aws ecs run-task`, poll until it stops, and return success or failure based on the exit code.

```groovy
pipeline {
    agent { label 'docker' }

    parameters {
        string(name: 'APP_BRANCH_NAME', defaultValue: 'origin/master')
        string(name: 'COMMAND', defaultValue: 'migrate --dry-run', description: 'Command to run in the container')
        choice(name: 'TF_ENV', choices: ['qa', 'production'])
    }

    environment {
        CLUSTER   = "app-cluster-${params.TF_ENV}"
        IMAGE_TAG = "${env.GIT_COMMIT[0..7]}"
    }

    stages {
        stage('Register task definition') {
            steps {
                dir('infrastructure/terraform/one-time-task') {
                    sh """
                        terraform init -input=false
                        terraform apply -input=false -auto-approve \
                            -var="app_version=${IMAGE_TAG}"
                    """
                }
            }
        }

        stage('Run task') {
            steps {
                script {
                    def taskArn = sh(
                        script: """
                            aws ecs run-task \
                                --cluster ${CLUSTER} \
                                --task-definition ${env.SERVICE_NAME}-one-time-task-${params.TF_ENV} \
                                --overrides '{"containerOverrides":[{"name":"app","command":${groovy.json.JsonOutput.toJson(params.COMMAND.split())}}]}' \
                                --query 'tasks[0].taskArn' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()

                    sh "aws ecs wait tasks-stopped --cluster ${CLUSTER} --tasks ${taskArn}"

                    def exitCode = sh(
                        script: """
                            aws ecs describe-tasks \
                                --cluster ${CLUSTER} \
                                --tasks ${taskArn} \
                                --query 'tasks[0].containers[0].exitCode' \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()

                    if (exitCode != '0') {
                        error("Task exited with code ${exitCode}")
                    }
                }
            }
        }
    }
}
```

`aws ecs wait tasks-stopped` blocks until the task stops. The exit code is then read from the task description to determine pass or fail.

### Jenkins job configuration

The job DSL for the standalone Jenkins job:

```groovy
pipelineJob("${team}/${repo}-standalone") {
    displayName("${repo} [Standalone]")

    logRotator { numToKeep(30) }
    throttleConcurrentBuilds { maxTotal(1) }

    parameters {
        stringParam('APP_BRANCH_NAME', 'origin/master', 'Branch or tag to deploy')
        choiceParam('TF_ENV', ['qa', 'production'])
        stringParam('COMMAND', 'migrate --dry-run', 'Command to run in the container')
    }

    definition {
        cpsScm {
            scm {
                git {
                    remote {
                        github("your-org/${repo}", 'ssh')
                        credentials('github-ssh-key')
                        branch('${APP_BRANCH_NAME}')
                    }
                    extensions { wipeOutWorkspace() }
                }
            }
            scriptPath("infrastructure/jenkins/standalone.groovy")
        }
    }
}
```

`throttleConcurrentBuilds { maxTotal(1) }` prevents two standalone tasks from running in parallel - important for migrations that must not run concurrently.

---

## Viewing Logs

Standalone task output is not visible in Jenkins - only the exit code is. To see what the task actually did, check Kibana and filter by service name and the time the task ran:

```
service: order-service AND NOT level: debug
```

Set the time range to cover when the Jenkins job ran. Because the standalone task uses the same Docker image and log driver as the main service, its output appears as regular log lines in the same index.

See [Kibana Log Browsing](../monitoring-js-tooling/kibana-log-browsing.md) for how to trace a specific task run.

---

## Scheduled Tasks: Recurring Commands via CloudWatch Events

For commands that need to run on a schedule - syncing data every hour, sending emails at 16:00 daily, running a nightly cleanup - use CloudWatch Events (EventBridge) to trigger an ECS task on a cron expression.

### The two target patterns

CloudWatch can trigger an ECS task directly, or it can publish a message to an SQS queue that the service reads.

**Direct ECS trigger** - CloudWatch starts a new container. Useful for heavy batch jobs where you want full container isolation and a clean exit code.

**SQS message trigger** - CloudWatch publishes a JSON event to a queue. The service's existing consumer picks it up and handles it like any other message. Simpler to implement when the service already has SQS consumers.

---

### Pattern 1: Direct ECS trigger

Define a task definition per cron job and a CloudWatch rule that runs it on a schedule. Using a list of jobs keeps the Terraform concise:

```hcl
locals {
  cron_jobs = [
    {
      name          = "sync-data"
      description   = "Sync external data every hour during business hours"
      command       = "sync-data"
      schedule_expr = var.sync_data_schedule
    },
    {
      name          = "send-notifications"
      description   = "Send scheduled notifications, daily at 16:00"
      command       = "send-notifications"
      schedule_expr = var.send_notifications_schedule
    },
  ]
}

resource "aws_ecs_task_definition" "cron_command_task" {
  count  = length(local.cron_jobs)
  family = "${var.service_name}-${local.cron_jobs[count.index].name}-${terraform.workspace}"

  container_definitions = templatefile("${path.module}/cron.json", {
    image_name            = var.image_name
    service_name          = var.service_name
    registry              = var.registry
    version               = var.app_version
    cpu                   = "200"
    memory                = "128"
    environment_variables = jsonencode(local.cron_env_variables)
    secrets               = jsonencode(var.secrets)
    command               = local.cron_jobs[count.index].command
  })

  execution_role_arn = local.cron_iam_role
  task_role_arn      = local.cron_iam_role
}

resource "aws_cloudwatch_event_rule" "cron_trigger" {
  count               = length(local.cron_jobs)
  name                = "${var.service_name}-${local.cron_jobs[count.index].name}-${terraform.workspace}"
  description         = local.cron_jobs[count.index].description
  schedule_expression = local.cron_jobs[count.index].schedule_expr
}

resource "aws_cloudwatch_event_target" "cron_target" {
  count     = length(local.cron_jobs)
  target_id = "${local.cron_jobs[count.index].name}-${terraform.workspace}"
  rule      = element(aws_cloudwatch_event_rule.cron_trigger.*.name, count.index)
  arn       = data.aws_ecs_cluster.apps.arn
  role_arn  = data.aws_iam_role.cloudwatch_ecs_role.arn

  ecs_target {
    launch_type         = "EC2"
    task_count          = 1
    task_definition_arn = element(aws_ecs_task_definition.cron_command_task.*.arn, count.index)
  }
}

data "aws_iam_role" "cloudwatch_ecs_role" {
  name = "cloudwatch-ecs-trigger-role"
}
```

The `cloudwatch-ecs-trigger-role` IAM role grants CloudWatch Events permission to call `ecs:RunTask` on the cluster. It needs a trust policy allowing `events.amazonaws.com` and a policy granting `ecs:RunTask` and `iam:PassRole`.

The `cron.json` template is simpler than the one-time task template: the `command` is a fixed value, not a runtime parameter:

```json
[
  {
    "name": "app",
    "image": "${registry}/${image_name}:${version}",
    "cpu": ${cpu},
    "memory": ${memory},
    "essential": true,
    "command": ["/app/run", "${command}"],
    "logConfiguration": {
      "logDriver": "json-file",
      "options": {
        "tag": "docker/${service_name}"
      }
    },
    "environment": ${environment_variables},
    "secrets": ${secrets}
  }
]
```

### schedule_expression syntax

The `schedule_expression` value can use either `rate()` or `cron()` format:

```
rate(1 hour)
rate(30 minutes)
cron(0 14 * * ? *)     # every day at 14:00 UTC
cron(0 9-19 * * ? *)   # every hour from 09:00 to 19:00 UTC
```

CloudWatch cron expressions always use UTC. Account for timezone offset when scheduling business-hours jobs.

The IAM trust policy for the CloudWatch trigger role:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "events.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

With a permissions policy granting `ecs:RunTask` on the target cluster and `iam:PassRole` on the task execution role.

---

### Pattern 2: CloudWatch to SQS

When the service already has SQS consumers, publish a trigger message to a queue instead of starting a new container:

```hcl
resource "aws_cloudwatch_event_rule" "cron_sync_trigger" {
  name                = "${var.service_name}-cron-sync-${terraform.workspace}"
  description         = "Trigger for data sync job"
  schedule_expression = var.cron_sync_schedule_expression
}

resource "aws_cloudwatch_event_target" "cron_sync_target" {
  target_id = "${var.service_name}-cron-sync-${terraform.workspace}"
  rule      = aws_cloudwatch_event_rule.cron_sync_trigger.name
  arn       = var.cron_event_queue_arn
  input     = file("${path.module}/templates/cron-event-sync.json")
}
```

The `input` is a static JSON file containing whatever message structure the consumer expects:

```json
{
  "type": "DataSync.Start"
}
```

The queue policy must allow `sqs:SendMessage` from `events.amazonaws.com`. The service consumer receives this on schedule and handles it alongside regular SQS messages. No new containers are started.

---

## Choosing Between the Two Task Types

**Use standalone (Jenkins-triggered) when:**
- Running a migration or backfill - you need to trigger it once at a specific point in a deployment
- You need a human to approve it first (the Jenkins job serves as the gate)
- You want to pass parameters at runtime (which command, which data range)

**Use scheduled (CloudWatch-triggered) when:**
- The job needs to run automatically on a recurring schedule
- It is a background sync, report generation, or cleanup that requires no human decision

**Use direct ECS target when:**
- The job is heavy (large memory/CPU) and needs a dedicated container
- You want ECS to report a clean exit code

**Use SQS target when:**
- The service already has SQS consumers and the job fits naturally as a message type
- The job is lightweight and you want to reuse existing consumer infrastructure

---

### For AI agents

```
Two ECS task patterns outside normal request handling:
1. Standalone (one-time): Terraform in infrastructure/terraform/one-time-task/ registers an ECS task definition. Jenkins pipeline runs it with aws ecs run-task + command override, then polls with aws ecs wait tasks-stopped. Check exit code from describe-tasks. Use for migrations and backfills. Logs visible in Kibana (not Jenkins).
2. Scheduled (CloudWatch): aws_cloudwatch_event_rule with schedule_expression (rate() or cron()) + aws_cloudwatch_event_target pointing to ECS cluster (direct container start) or SQS queue (message to existing consumer). CloudWatch needs an IAM role with ecs:RunTask + iam:PassRole to trigger ECS directly. CloudWatch schedule_expression uses UTC.
```

Reference: `https://michalsniezko.github.io/devops-infrastructure-cicd/ecs-standalone-scheduled-tasks.html`
