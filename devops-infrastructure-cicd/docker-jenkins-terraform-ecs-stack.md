---
layout: default
title: The CI/CD Stack
parent: Infrastructure & CI/CD
nav_order: 10
---

## The CI/CD Stack: Docker, Jenkins, Terraform, and ECS

Five tools cover every step from writing code to running it in production:

| Tool | Role |
|---|---|
| **Docker** | Packages the application into a portable, reproducible image |
| **Makefile** | Wraps long commands into short targets for both local development and CI |
| **Jenkins** | Orchestrates the pipeline: build, test, push, deploy |
| **Terraform** | Declares the AWS infrastructure that runs the application |
| **AWS ECS** | Schedules and runs the containers in production |

Each tool owns one layer. They connect in sequence: Docker builds the artifact. Make calls Docker. Jenkins calls Make. Terraform provisions what ECS needs. ECS pulls the image Docker produced and runs it.

---

## Docker: Packaging the Application

Docker turns the application and all its dependencies into an image that runs identically in CI, staging, and production.

### Multi-stage Dockerfile for a PHP service

```dockerfile
# Stage 1: install dependencies (requires COMPOSER_AUTH for private packages)
FROM composer:2 AS builder
WORKDIR /app
COPY composer.json composer.lock ./
RUN --mount=type=secret,id=composer_auth,target=/root/.composer/auth.json \
    composer install --no-dev --no-scripts --prefer-dist

# Stage 2: production image
FROM php:8.3-fpm-alpine AS production
WORKDIR /app

RUN apk add --no-cache nginx supervisor && \
    docker-php-ext-install pdo_pgsql opcache

COPY --from=builder /app/vendor ./vendor
COPY . .
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

RUN chmod -R 755 /app && chown -R www-data:www-data /app/var

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

The builder stage installs Composer dependencies and has access to the private auth token. The production stage copies only the installed packages - the auth token never ends up in the final layer.

### docker-compose for local development and CI

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: production
    ports:
      - "8080:80"
    environment:
      APP_ENV: dev
      DATABASE_URL: postgresql://app:secret@db:5432/app
    volumes:
      - .:/app  # mount source for live reload in dev

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  wiremock:
    image: wiremock/wiremock:latest
    ports:
      - "8081:8080"
    volumes:
      - ./tests/stubs:/home/wiremock/mappings
```

```yaml
# docker-compose.ci.yml - lighter version for CI (no source mount, no exposed ports)
services:
  app:
    build:
      context: .
      target: production
    environment:
      APP_ENV: test
      DATABASE_URL: postgresql://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5

  wiremock:
    image: wiremock/wiremock:latest
    volumes:
      - ./tests/stubs:/home/wiremock/mappings
```

Using `service_healthy` on the `db` dependency prevents tests from running before PostgreSQL is accepting connections - a common source of flaky CI failures.

---

## Makefile: Short Commands for Long Operations

`make` targets serve two purposes: they give developers short commands for daily work, and they give Jenkins a stable interface that doesn't need to know the underlying implementation.

```makefile
# Makefile
SERVICE_NAME  := order-service
ECR_REGISTRY  := 123456789012.dkr.ecr.eu-west-1.amazonaws.com
IMAGE_TAG     ?= $(shell git rev-parse --short HEAD)
ENV           ?= qa

.PHONY: build test push deploy up down logs shell

## Local development
up:
	docker compose up -d

down:
	docker compose down -v

logs:
	docker compose logs -f app

shell:
	docker compose exec app sh

## CI targets
build:
	docker build \
		--target production \
		--build-arg BUILDKIT_INLINE_CACHE=1 \
		--cache-from $(ECR_REGISTRY)/$(SERVICE_NAME):latest \
		-t $(ECR_REGISTRY)/$(SERVICE_NAME):$(IMAGE_TAG) .

test:
	docker compose -f docker-compose.ci.yml run --rm app \
		php vendor/bin/phpunit
	docker compose -f docker-compose.ci.yml down -v

push:
	aws ecr get-login-password --region eu-west-1 \
		| docker login --username AWS --password-stdin $(ECR_REGISTRY)
	docker push $(ECR_REGISTRY)/$(SERVICE_NAME):$(IMAGE_TAG)
	docker tag $(ECR_REGISTRY)/$(SERVICE_NAME):$(IMAGE_TAG) \
		$(ECR_REGISTRY)/$(SERVICE_NAME):latest
	docker push $(ECR_REGISTRY)/$(SERVICE_NAME):latest

deploy:
	cd terraform/environments/$(ENV) && \
		terraform init -input=false && \
		terraform apply -input=false -auto-approve \
			-var="image_tag=$(IMAGE_TAG)"
```

Jenkins then calls `make build`, `make test`, `make push`, `make deploy` - four clean commands. If the underlying implementation changes (different registry, different test command), Jenkins doesn't change.

The `IMAGE_TAG ?=` pattern uses the git commit SHA by default but allows Jenkins to override it: `make build IMAGE_TAG=$GIT_COMMIT`.

---

## Jenkins: Orchestrating the Pipeline

Jenkins calls Make targets in sequence. Each stage gates the next - a test failure prevents the push, a push failure prevents the deploy.

```groovy
pipeline {
    agent { label 'docker' }

    environment {
        ECR_REGISTRY = '123456789012.dkr.ecr.eu-west-1.amazonaws.com'
        SERVICE_NAME = 'order-service'
        IMAGE_TAG    = "${env.GIT_COMMIT[0..7]}"
    }

    stages {
        stage('Build') {
            steps {
                sh "make build IMAGE_TAG=${IMAGE_TAG}"
            }
        }

        stage('Test') {
            steps {
                sh "make test IMAGE_TAG=${IMAGE_TAG}"
            }
            post {
                always {
                    // Collect test reports regardless of pass/fail
                    junit allowEmptyResults: true, testResults: 'build/reports/**/*.xml'
                }
            }
        }

        stage('Push') {
            when { branch 'main' }
            steps {
                sh "make push IMAGE_TAG=${IMAGE_TAG}"
            }
        }

        stage('Deploy QA') {
            when { branch 'main' }
            steps {
                sh "make deploy ENV=qa IMAGE_TAG=${IMAGE_TAG}"
                sh """
                    aws ecs wait services-stable \
                        --cluster qa \
                        --services ${SERVICE_NAME} \
                        --region eu-west-1
                """
                sh "./scripts/smoke-test.sh https://qa.order-service.internal"
            }
        }

        stage('Deploy Production') {
            when { branch 'main' }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                sh "make deploy ENV=production IMAGE_TAG=${IMAGE_TAG}"
                sh """
                    aws ecs wait services-stable \
                        --cluster production \
                        --services ${SERVICE_NAME} \
                        --region eu-west-1
                """
            }
        }
    }

    post {
        failure {
            slackSend channel: '#deployments',
                color: 'danger',
                message: "Deploy failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

`aws ecs wait services-stable` blocks until all tasks in the service are healthy or times out. Running smoke tests after the wait - not before - is what actually verifies the deployment succeeded. For more on deployment strategies and rollback, see [Zero-Downtime Deployments](zero-downtime-deployments.md).

---

## Terraform: Declaring the Infrastructure

Terraform describes what AWS resources should exist. It doesn't run the application - it creates and configures the scaffolding that ECS needs to run it.

A minimal ECS setup requires: an ECR repository (stores Docker images), an ECS cluster and task definition (what to run and with what config), an ECS service (how many copies to keep running), and an ALB with a target group (routes traffic in).

### ECR repository

```hcl
# terraform/modules/service/ecr.tf
resource "aws_ecr_repository" "service" {
  name                 = var.service_name
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }

  lifecycle_policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 20 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 20
      }
      action = { type = "expire" }
    }]
  })
}
```

### ECS task definition

```hcl
# terraform/modules/service/ecs.tf
resource "aws_ecs_task_definition" "service" {
  family                   = var.service_name
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = var.service_name
    image = "${aws_ecr_repository.service.repository_url}:${var.image_tag}"

    portMappings = [{
      containerPort = 80
      protocol      = "tcp"
    }]

    environment = [
      { name = "APP_ENV", value = var.environment }
    ]

    secrets = [
      {
        name      = "DATABASE_URL"
        valueFrom = "/${var.environment}/${var.service_name}/database_url"
      }
    ]

    healthCheck = {
      command     = ["CMD-SHELL", "curl -sf http://localhost/health/live || exit 1"]
      interval    = 10
      timeout     = 5
      retries     = 3
      startPeriod = 30
    }

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = "/ecs/${var.environment}/${var.service_name}"
        awslogs-region        = var.aws_region
        awslogs-stream-prefix = "ecs"
      }
    }
  }])

  stop_timeout = 60
}
```

`secrets` pulls values from AWS SSM Parameter Store at task startup - the container receives them as environment variables but the values are never stored in Terraform state or the task definition JSON. See [AWS SSM Parameter Store](aws-ssm-parameter-store.md) for how to store those secrets.

### ECS service

```hcl
resource "aws_ecs_service" "service" {
  name            = var.service_name
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.service.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.service.arn
    container_name   = var.service_name
    container_port   = 80
  }

  deployment_minimum_healthy_percent = 50
  deployment_maximum_percent         = 200

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  lifecycle {
    # Prevent Terraform from reverting manual scaling adjustments
    ignore_changes = [desired_count]
  }
}
```

### Variables and environments

```hcl
# terraform/modules/service/variables.tf
variable "service_name" { type = string }
variable "environment"  { type = string }
variable "image_tag"    { type = string }   # injected by Jenkins: -var="image_tag=abc1234"
variable "desired_count" {
  type    = number
  default = 2
}
```

```
# terraform/environments/qa/
├── main.tf          # calls the module
├── qa.tfvars        # environment-specific values
└── backend.tf       # S3 remote state config
```

```hcl
# terraform/environments/qa/main.tf
module "order_service" {
  source       = "../../modules/service"
  service_name = "order-service"
  environment  = "qa"
  image_tag    = var.image_tag   # passed in by make deploy
}
```

---

## AWS ECS: Running the Containers

ECS is the scheduler. Given a task definition (what to run) and a service (how many to keep running), it handles placement, health monitoring, and replacement of failed tasks.

### Key concepts

**Cluster** - a logical grouping of compute capacity. On Fargate, the cluster is just a name - AWS manages the underlying hosts.

**Task definition** - a versioned blueprint: which Docker image, how much CPU/memory, environment variables, secrets, health check config, log routing. Each `terraform apply` with a new `image_tag` registers a new revision.

**Service** - keeps `desired_count` tasks running at all times. If a task fails its health check, ECS replaces it. During a deployment, ECS replaces tasks according to `minimum_healthy_percent` and `maximum_percent`.

**Fargate** - the serverless ECS launch type. You define CPU and memory per task; AWS provisions the host. No EC2 instances to manage or patch.

### How a deployment flows

```
1. make push        → new image pushed to ECR as :abc1234
2. make deploy      → Terraform registers new task definition revision
                      (same config, new image_tag)
3. Terraform calls  → aws ecs update-service with new task def ARN
4. ECS              → starts new tasks (new image)
5. ALB              → health checks new tasks on /health/ready
6. After threshold  → routes traffic to new tasks
7. ECS              → deregisters and drains old tasks
8. aws ecs wait     → Jenkins blocks until stable
9. Smoke tests      → verify the new version actually works
```

The circuit breaker (`rollback = true`) watches the deployment. If new tasks consistently fail health checks, ECS automatically reverts the service to the previous task definition.

---

## How the Pieces Fit Together

```
Developer machine                Jenkins                     AWS
-----------------                -------                     ---
make up          (dev loop)
make test        (local verify)
                                 git push triggers build
                                 make build  ──────────────> (Docker build)
                                 make test   ──────────────> (docker compose ci)
                                 make push   ──────────────> ECR
                                 make deploy ──────────────> Terraform
                                                             └─> ECS task def (new revision)
                                                             └─> ECS service update
                                 ecs wait services-stable <── (polls ECS)
                                 smoke-test.sh             <── (hit /health/ready)
```

The Makefile is the shared vocabulary: the same `make test` command runs locally and in CI. Adding a new test suite or switching from PHPUnit to Pest requires changing one line in the Makefile - not in Jenkins.

---

### For AI agents

```
Deployment stack: Docker builds the image (multi-stage, Composer secrets in builder stage only). Makefile wraps all commands (make build/test/push/deploy). Jenkins calls make targets - never raw docker/terraform commands. Terraform registers a new ECS task definition revision on each deploy by passing -var="image_tag=<sha>". ECS Fargate runs desired_count tasks; circuit breaker auto-rollback on health check failure. Always run aws ecs wait services-stable before declaring deploy successful.
```

Reference: `https://michalsniezko.github.io/devops-infrastructure-cicd/docker-jenkins-terraform-ecs-stack.html`
