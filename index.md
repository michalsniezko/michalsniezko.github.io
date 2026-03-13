---
layout: default
title: Home
nav_order: 1
description: "Start here for distributed systems patterns."
permalink: /
---

# The Distributed Engineer's Handbook

> Battle-tested patterns, architectural decisions, and sharp lessons from building distributed systems that actually run in production.
> 
> No theory without context - every entry here came from a real problem in a PHP/AWS/microservices stack.

---

## Contents

### 1. Messaging & Event-Driven Architecture

SNS fan-out, SQS consumers, filter policies, adaptive message splitting, and subscription-based routing.

- [Messaging & Event-Driven Architecture](aws-event-driven/index.md)

### 2. Microservices & Service Design

How services find each other with Consul, how requests get traced with Zipkin B3 headers, stale cache fallbacks for resilience, and why your API docs belong in the code.

- [Microservices & Service Design](microservices-observability/index.md)

### 3. PHP Patterns in Practice

Generators, DTOs, processor chains with circuit breakers, event replay, specification pattern, and pluggable validator pipelines.

- [PHP Patterns in Practice](backend-patterns-optimization/index.md)

### 4. Database & Persistence

Normal forms, TOAST and WAL internals, the Data Mapper pattern, Doctrine JSON patch changelogs, and upserts that eliminate race conditions.

- [Database & Persistence](database-patterns/index.md)

### 5. Testing & Concurrency

Distributed locks, WireMock for external API simulation, PHPUnit mocking with Prophecy, and concurrency stress testing.

- [Testing & Concurrency](testing-concurrency-locks/index.md)

### 6. Infrastructure & CI/CD

Terraform with remote state locking, Jenkins pipelines that apply the exact plan you reviewed, and secrets that never touch a Git repo.

- [Infrastructure & CI/CD](devops-infrastructure-cicd/index.md)

### 7. Autoscaling & Lambda

Autoscaling ECS workers based on SQS queue depth instead of CPU, CloudWatch alarms that catch stale consumers, and Lambda concurrency guardrails.

- [Autoscaling & Lambda](scaling-cloudwatch-autoscaling/index.md)

### 8. Self-Hosted Monitoring

The TICK stack (Telegraf, InfluxDB, Chronograf, Kapacitor), Telegram alerts, and custom metric pipelines.

- [Self-Hosted Monitoring](monitoring-js-tooling/index.md)

### 9. JavaScript & Frontend Tooling

Rollup bundler configs, frontend dictionary sync from backend DTOs, and the wrap interceptor pattern for Node.js Lambdas.

- [JavaScript & Frontend Tooling](js-frontend-tooling/index.md)

---

## Code Examples

Articles use whichever language fits the context:

- **PHP:** domain logic, Symfony services, Doctrine, Monolog processors
- **JavaScript / Node.js:** Lambda handlers, AWS SDK v3, Rollup configs
- **HCL (Terraform):** infrastructure resources and remote state
- **SQL:** PostgreSQL upserts and conflict handling
- **YAML / TOML:** Symfony config, Telegraf/Kapacitor configs
- **Bash:** CI scripts, `gh` CLI workflows, Kapacitor task loading

Most articles mix languages - e.g. a PHP publisher paired with the Terraform that deploys the SNS topic it publishes to.

---

## About Me

I'm Michał Śnieżko, a backend software engineer at **Auto1 Group** in Kraków, Poland, with 10+ years of experience building web applications in PHP.

I work in a microservice environment where I integrate services through internal clients, Consul discovery, and AWS messaging (SNS/SQS). Day to day I build event pipelines with filtering and ordering safeguards, set up distributed tracing with Zipkin and Kibana, manage infrastructure with Terraform, and run CI/CD through Jenkins. I optimize PHP code using patterns like Data Mapper, generators, and bulk loading, and I test it with PHPUnit, WireMock, and concurrency stress scripts.

Previously at **Codibly** (5 years), I worked on Symfony/PHP 8 backends for insurance clients, built CQRS-based systems with message queues, and consulted on e-mobility solutions (OCPI/OCPP protocols, EV charging interoperability). Before that I built REST APIs and microservices at **Ailleron** in the fintech space.

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

- [GitHub](https://github.com/michalsniezko)
- [LinkedIn](https://linkedin.com/in/michal-sniezko)
