# The Distributed Engineer's TIL

> Battle-tested patterns, architectural decisions, and sharp lessons from building distributed systems that actually run in production. No theory without context — every entry here came from a real problem in a PHP/AWS/microservices stack.

---

## Contents

### 1. Event-Driven Architecture (AWS)

Decoupling services with SNS fan-out, SQS consumers, and filter policies that keep queues lean.

- [AWS Messaging Patterns](./aws-messaging-patterns.md)

### 2. Microservices & Observability

How services find each other with Consul, how requests get traced with Zipkin B3 headers, and why your API docs belong in the code.

- [Microservices & Observability](./microservices-observability.md)

### 3. High-Performance Backend (PHP)

Generators that process millions of rows in 2MB of memory, bulk-loading patterns that kill N+1 calls, and structured DTOs for frontend consumption.

- [Backend Patterns & Optimization](./backend-patterns-optimization.md)

### 4. Concurrency & Resilience

Distributed locks that prevent duplicate invoices, upserts that eliminate race conditions, and bash scripts that prove your fix works under real concurrency.

- [Testing & Concurrency](./testing-concurrency-locks.md)

### 5. Infrastructure as Code & CI/CD

Terraform with remote state locking, Jenkins pipelines that apply the exact plan you reviewed, and secrets that never touch a Git repo.

- [DevOps & Infrastructure](./devops-infrastructure-cicd.md)

### 6. Cloud-Native Scaling

Autoscaling ECS workers based on SQS queue depth instead of CPU, CloudWatch alarms that catch stale consumers, and Lambda concurrency guardrails.

- [Scaling & CloudWatch](./scaling-cloudwatch-autoscaling.md)

### 7. Monitoring & Tooling

Self-hosted monitoring with the TICK stack, Telegram alerts that wake you at 3 AM, and Rollup configs that produce clean library bundles.

- [Monitoring & JS Tooling](./monitoring-js-tooling.md)

---

## Tech Stack

![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat-square&logo=amazonwebservices&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-777BB4?style=flat-square&logo=php&logoColor=white)
![Symfony](https://img.shields.io/badge/Symfony-000000?style=flat-square&logo=symfony&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)

---

## Connect

- [GitHub](https://github.com/your-username)
- [LinkedIn](https://linkedin.com/in/your-profile)
