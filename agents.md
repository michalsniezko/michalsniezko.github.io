---
layout: default
title: For AI Agents
nav_order: 11
---

# For AI Agents

This page is a source of truth for AI coding assistants. Each snippet can be pasted into a `CLAUDE.md` or system prompt to guide an agent on the patterns and conventions documented in this handbook.

Two formats are available:
- **Inline rule** - self-contained, works without network access, paste directly
- **Reference link** - tells the agent to fetch the full article for richer context

---

## URL-Only Quick Start

Paste this block into `CLAUDE.md`. The agent fetches each page on demand when the topic comes up:

```
## Engineering References
When working on the following topics, read the linked article before writing code:

# Database
- PostgreSQL index migrations (CONCURRENTLY): https://michalsniezko.github.io/database-patterns/concurrent-index-migrations.html
- Race conditions and upserts: https://michalsniezko.github.io/database-patterns/race-conditions-upsert.html
- Lazy DB result streaming: https://michalsniezko.github.io/database-patterns/lazy-db-streaming-generators.html
- Data Mapper with bulk loading: https://michalsniezko.github.io/database-patterns/data-mapper-bulk-loading.html
- JSON patch changelogs: https://michalsniezko.github.io/database-patterns/json-patch-changelog.html
- PostgreSQL TOAST and WAL: https://michalsniezko.github.io/database-patterns/postgres-toast-wal.html
- Active Record vs Data Mapper: https://michalsniezko.github.io/database-patterns/active-record-vs-data-mapper.html
- Database-backed task queue: https://michalsniezko.github.io/backend-patterns-optimization/database-backed-task-queue.html

# PHP Patterns
- Generator pipelines: https://michalsniezko.github.io/backend-patterns-optimization/generator-patterns.html
- Specification pattern: https://michalsniezko.github.io/backend-patterns-optimization/specification-pattern.html
- Pluggable validator pipeline: https://michalsniezko.github.io/backend-patterns-optimization/pluggable-validator-pipeline.html
- Processor chain with error accumulation: https://michalsniezko.github.io/backend-patterns-optimization/processor-chain-error-accumulation.html
- Event sourcing: https://michalsniezko.github.io/backend-patterns-optimization/event-sourcing.html
- Event replay from changelogs: https://michalsniezko.github.io/backend-patterns-optimization/changelog-reconstruction-replay.html

# Testing
- Test types (unit/integration/functional/API): https://michalsniezko.github.io/testing-concurrency-locks/test-types.html
- JS testing with Jest (DI mocking): https://michalsniezko.github.io/testing-concurrency-locks/jest-testing.html
- PHP mocking with Prophecy: https://michalsniezko.github.io/testing-concurrency-locks/prophecy-promise-pattern.html
- WireMock for endpoint testing: https://michalsniezko.github.io/testing-concurrency-locks/wiremock-endpoint-testing.html
- Distributed lock: https://michalsniezko.github.io/testing-concurrency-locks/distributed-lock.html
- Out-of-order message testing: https://michalsniezko.github.io/testing-concurrency-locks/out-of-order-message-testing.html

# Microservices
- Repositories as service clients: https://michalsniezko.github.io/microservices-observability/repositories-service-clients.html
- Distributed tracing with B3 headers: https://michalsniezko.github.io/microservices-observability/distributed-tracing-b3.html
- Monolog Zipkin processor: https://michalsniezko.github.io/microservices-observability/monolog-zipkin-processor.html
- OAuth2 for service-to-service: https://michalsniezko.github.io/microservices-observability/oauth2.html
- Stale cache fallback: https://michalsniezko.github.io/microservices-observability/stale-cache-fallback.html

# AWS & Infrastructure
- SNS fan-out to SQS: https://michalsniezko.github.io/aws-event-driven/sns-fanout.html
- SNS filter policies: https://michalsniezko.github.io/aws-event-driven/sns-filters.html
- Out-of-order message handling: https://michalsniezko.github.io/aws-event-driven/out-of-order.html
- Adaptive message splitting: https://michalsniezko.github.io/aws-event-driven/adaptive-message-splitting.html
- SQS-based autoscaling: https://michalsniezko.github.io/scaling-cloudwatch-autoscaling/sqs-based-autoscaling.html
- AWS SDK pitfalls: https://michalsniezko.github.io/scaling-cloudwatch-autoscaling/aws-sdk-pitfalls.html
- Terraform project structure: https://michalsniezko.github.io/devops-infrastructure-cicd/terraform-project-structure.html
- Terraform required variables: https://michalsniezko.github.io/devops-infrastructure-cicd/terraform-required-variable-jenkins.html
- Secrets with SSM Parameter Store: https://michalsniezko.github.io/devops-infrastructure-cicd/aws-ssm-parameter-store.html

# Frontend & JavaScript
- Effector state management: https://michalsniezko.github.io/js-frontend-tooling/effector.html
- Rollup/esbuild for libraries: https://michalsniezko.github.io/js-frontend-tooling/rollup-bundler.html
- Wrap interceptor (Lambda): https://michalsniezko.github.io/js-frontend-tooling/wrap-interceptor-pattern.html

# Observability
- Kibana log browsing and trace IDs: https://michalsniezko.github.io/monitoring-js-tooling/kibana-log-browsing.html
- ELK stack setup: https://michalsniezko.github.io/monitoring-js-tooling/elk-stack.html
```

---

## Full Inline Rules

Paste this block to embed all rules directly without network fetches:

```
## Engineering Conventions

### Database
- PostgreSQL production indexes: always use CREATE INDEX CONCURRENTLY to avoid write locks. Disable transaction wrapper in Doctrine migrations with isTransactional(): false. On build failure, check pg_index.indisvalid and DROP INDEX CONCURRENTLY before retrying.
- Upserts: replace SELECT-then-INSERT with INSERT ... ON CONFLICT (unique_field) DO UPDATE SET ... WHERE updated_at to eliminate race conditions and prevent stale overwrites.
- Large result sets: never fetchAllAssociative() on unbounded queries. Wrap PDOStatement in a generator (foreach $stmt as $row { yield $row; }) to stream one row at a time. Memory stays flat.
- Bulk hydration: collect all IDs first, make one bulk fetch, index results by ID, map back. Never query inside a loop.
- JSON column history: compute RFC 6902 forward and reverse patches on every change, store in a changelog table. Store reverse patches to support undo without full snapshots.
- Async tasks: use a DB table with status enum (0=QUEUE, 1=SENT, 2=FAILED). Enqueue inside the same transaction as the business operation. Add partial index WHERE status = 0. Add retry_count for backoff logic.

### PHP Patterns
- Memory-intensive pipelines: use PHP generators with yield from. Memory stays ~2MB regardless of dataset size. Trade-off: forward-only, single-pass iteration.
- Business rules: extract each rule into a Specification class with isSatisfied(): bool. Compose in a service. Never embed multiple business conditions in a single if block.
- Validation: use Symfony tagged services for pluggable, context-specific validator pools.
- Processor chains: pass a mutable error-accumulation DTO through processors. Throw for hard failures, accumulate soft errors. Log accumulated errors after the chain.
- DI: always constructor injection. Never pull from container in business logic.

### Testing
- Unit: mock the injected dependency, never the transport layer (global.fetch, PDO, Guzzle). No real I/O.
- Integration: hit a real database. Use transaction rollback in setUp/tearDown for cleanup. Stub external HTTP with WireMock.
- Functional: nothing mocked. Assert both HTTP response and DB state after the request.
- API: stub downstream services with WireMock. Verify status codes, response shapes, and error handling.
- Effector: use fork({ handlers }) and allSettled() for isolated, framework-free store tests.

### Microservices
- HTTP repositories: configure Symfony scoped client with base_uri in framework.yaml. Inject the named client. Use relative URLs in the repository. Never inject base URLs as constructor strings.
- Distributed tracing: propagate X-B3-TraceId, X-B3-SpanId, X-B3-ParentSpanId on every outbound HTTP call. Inject traceId into every Monolog log entry via a processor. A single missing forward breaks the entire trace chain.
- Service auth: OAuth2 Client Credentials for service-to-service. Cache token in Redis/APCu with TTL shorter than expiry. Never fetch a new token per request.
- Resilience: serve stale cache entries when upstream fails. Store without TTL, track expiry manually. Always log when serving stale data.

### AWS
- SNS: publish once to a topic; each consumer owns its SQS queue and subscription. Filter on the subscription side, not in the publisher.
- Out-of-order: store event_timestamp on every message. At the consumer, discard events older than last_processed_at.
- Autoscaling: scale ECS on BacklogPerInstance = queue_depth / running_tasks, not CPU.
- SDK clients: initialize once at module level, outside the Lambda handler. Never instantiate inside the handler body.
- Terraform variables: always add a default to prevent unattended Jenkins runs from waiting for interactive input.
- Secrets: SSM Parameter Store as SecureString, referenced by ARN. Never in Git or plain env vars.

### Frontend
- Effector: prefix stores with $, effects with Fx. Use sample() for declarative unit wiring. Test with fork()/allSettled(). Never mutate store state directly.
- Libraries: list all peerDependencies in bundler external[]. Ship ESM + CJS. Prefer esbuild for new projects.
- Lambda handlers: wrap with a higher-order interceptor for logging, auth, error formatting. Keep business logic pure.

### Observability
- Every log entry must include traceId from X-B3-TraceId. In Kibana, filter extra.traceId: "<id>" to see the full cross-service request journey.
- KQL: field:value, AND/OR/NOT, ranges (field > n), wildcards (field: prefix*), exact phrases in quotes, NOT field: * for missing fields.
- Alerts: use state-change-only flags in Kapacitor TICKscripts to suppress repeat alerts on sustained threshold breaches.
```

---

## Per-Article Snippets

### Database & Persistence

**[Index Migrations - Normal vs CONCURRENTLY](https://michalsniezko.github.io/database-patterns/concurrent-index-migrations.html)**
```
When adding indexes to production PostgreSQL tables: always use CREATE INDEX CONCURRENTLY to avoid write locks. Accepts 2-3x longer build time in exchange for zero downtime. In Doctrine migrations, set isTransactional(): false since CONCURRENTLY cannot run inside a transaction. If a build fails, check for invalid indexes with pg_index.indisvalid = false and DROP INDEX CONCURRENTLY before retrying.
```

**[Race Conditions and Upsert](https://michalsniezko.github.io/database-patterns/race-conditions-upsert.html)**
```
Never use SELECT-then-INSERT for writes that can race. Use INSERT ... ON CONFLICT (unique_column) DO UPDATE SET ... WHERE to make the upsert atomic at the database level. Add WHERE old_updated_at < EXCLUDED.updated_at to prevent stale data from overwriting newer records.
```

**[Lazy DB Result Streaming](https://michalsniezko.github.io/database-patterns/lazy-db-streaming-generators.html)**
```
Never call fetchAllAssociative() on a query that can return a large or unbounded result set. Wrap PDOStatement in a generator (foreach $stmt as $row { yield $row; }) to stream one row at a time. Memory stays flat regardless of row count. Trade-off: forward-only cursor, single-pass, holds DB connection open for the duration.
```

**[Data Mapper with Bulk Loading](https://michalsniezko.github.io/database-patterns/data-mapper-bulk-loading.html)**
```
Never query or call an API inside a loop (N+1). Collect all unique IDs first, make one bulk fetch, index results by ID into an associative array, then map back to objects. Reduces N queries to 1 regardless of collection size.
```

**[JSON Patch Changelog](https://michalsniezko.github.io/database-patterns/json-patch-changelog.html)**
```
For auditable JSON columns: compute RFC 6902 patches on every change and store both forward and reverse patches in a changelog table. The reverse patch enables undo without storing full snapshots. Use a diffing library, not string comparison.
```

**[PostgreSQL TOAST and WAL](https://michalsniezko.github.io/database-patterns/postgres-toast-wal.html)**
```
Avoid SELECT * on tables with large JSONB or TEXT columns - TOAST decompresses every value even if unused. Updating a large JSONB column writes the entire value to WAL. For frequently-updated large values, store only a reference key in the DB and the payload in S3.
```

**[Database-Backed Task Queue](https://michalsniezko.github.io/backend-patterns-optimization/database-backed-task-queue.html)**
```
For transactional async tasks: use a DB table with a status enum (0=QUEUE, 1=SENT, 2=FAILED) instead of an external broker. Enqueue inside the same transaction as the triggering business operation. Add a partial index WHERE status = 0 for the worker poll query. Add retry_count for exponential backoff.
```

---

### PHP Patterns

**[Generator Patterns](https://michalsniezko.github.io/backend-patterns-optimization/generator-patterns.html)**
```
For memory-intensive data pipelines: use PHP generators with yield from instead of array_map/array_filter. Memory stays flat (~2MB) regardless of dataset size. yield from delegates to sub-generators cleanly. Trade-off: forward-only, no random access, single pass.
```

**[Specification Pattern](https://michalsniezko.github.io/backend-patterns-optimization/specification-pattern.html)**
```
Extract each business rule into its own Specification class with isSatisfied(Context $ctx): bool. Compose specifications in a service. Never embed multiple business conditions in a single method. Each specification is independently unit-testable and named after the domain concept.
```

**[Pluggable Validator Pipeline](https://michalsniezko.github.io/backend-patterns-optimization/pluggable-validator-pipeline.html)**
```
For context-specific validation: tag validator services with Symfony tags and inject them into context-specific pools via tagged iterators. Run only the validators relevant to the current context. New validators are added without modifying existing code.
```

**[Processor Chain](https://michalsniezko.github.io/backend-patterns-optimization/processor-chain-error-accumulation.html)**
```
For multi-step processing with partial failures: pass a mutable result DTO through a chain of processors. Throw domain exceptions for hard stops. Accumulate soft errors in the DTO and continue processing. Log all accumulated errors after the chain completes.
```

**[Event Sourcing](https://michalsniezko.github.io/backend-patterns-optimization/event-sourcing.html)**
```
For auditable state: store every change as an immutable event (type, payload, timestamp) in an append-only table. Never update or delete events. Derive current state by replaying. Keep read model projections separate from the write model.
```

**[Event Replay from Changelog](https://michalsniezko.github.io/backend-patterns-optimization/changelog-reconstruction-replay.html)**
```
To reconstruct historical state: fetch changelog rows up to the target timestamp and replay through a registry of Strategy objects keyed by change type. Unknown change types are skipped. Avoids encoding all replay logic in a single monolithic function.
```

---

### Testing

**[Test Types](https://michalsniezko.github.io/testing-concurrency-locks/test-types.html)**
```
Four types: Unit (single class, mock injected deps via DI, no I/O), Integration (real DB/queue, stub external HTTP with WireMock, transaction rollback for cleanup), Functional (full HTTP flow, real DB, nothing mocked, assert DB state), API (HTTP contract, stub downstream services, verify status codes and response shapes). Never call a test "unit" if it touches the network, filesystem, or database.
```

**[JS Testing with Jest](https://michalsniezko.github.io/testing-concurrency-locks/jest-testing.html)**
```
In Jest unit tests: never mock global.fetch or axios directly. Define an ApiClient interface, inject it via the constructor, and mock the interface in tests. The test controls exactly what the dependency returns without coupling to HTTP internals. No global state patched, no afterEach cleanup needed.
```

**[PHP Mocking with Prophecy](https://michalsniezko.github.io/testing-concurrency-locks/prophecy-promise-pattern.html)**
```
In PHPUnit with Prophecy: prophesize() an interface, chain willReturn()/shouldBeCalled()/shouldNotBeCalled() on method predictions, reveal() before passing to the constructor. Never mock concrete classes when an interface exists. Always inject the mock via the constructor, never via setters or property access.
```

**[WireMock for Endpoint Testing](https://michalsniezko.github.io/testing-concurrency-locks/wiremock-endpoint-testing.html)**
```
For integration tests that call upstream services: run WireMock as a Docker container. Define stubs in tests/wiremock/mappings/ JSON files. Call POST /__admin/reset in setUp() when tests share a WireMock instance to prevent stub leakage. Use scenarioName for stateful multi-step sequences.
```

**[Distributed Lock](https://michalsniezko.github.io/testing-concurrency-locks/distributed-lock.html)**
```
For operations that must not run concurrently: use Symfony Lock with a Redis or database store. Set a TTL as a safety net for dead processes. Always release in a finally block. Acquire before reading, not just before writing.
```

**[Out-of-Order Message Testing](https://michalsniezko.github.io/testing-concurrency-locks/out-of-order-message-testing.html)**
```
When testing SQS consumers that must handle out-of-order delivery: write tests that send messages in reverse or random order and assert that only the latest state is applied. Verify that older messages are discarded, not applied on top of newer ones.
```

---

### Microservices

**[Repositories as Service Clients](https://michalsniezko.github.io/microservices-observability/repositories-service-clients.html)**
```
For HTTP calls to upstream services: configure a Symfony scoped client with base_uri in framework.yaml. Inject the pre-configured client into the repository by name. Use relative URLs in the repository. Never inject base URLs as constructor string arguments.
```

**[Distributed Tracing with B3 Headers](https://michalsniezko.github.io/microservices-observability/distributed-tracing-b3.html)**
```
On every outbound HTTP call: forward X-B3-TraceId, X-B3-SpanId, X-B3-ParentSpanId, X-B3-Sampled from the incoming request. Generate a new SpanId for the outbound span and set ParentSpanId to the caller's SpanId. Dropping these headers in any service breaks the trace chain for all downstream services.
```

**[Monolog Zipkin Processor](https://michalsniezko.github.io/microservices-observability/monolog-zipkin-processor.html)**
```
Add a Monolog processor that reads X-B3-TraceId and X-B3-SpanId from RequestStack and adds them as extra.traceId and extra.spanId to every log record. Register in monolog.yaml. For CLI/worker contexts where RequestStack returns null, generate a synthetic trace ID rather than logging 'no-trace'.
```

**[OAuth2 Service-to-Service](https://michalsniezko.github.io/microservices-observability/oauth2.html)**
```
For service-to-service auth: use OAuth2 Client Credentials flow. Cache the access token in Redis or APCu with a TTL slightly shorter than its expiry. Never fetch a new token on every request. Attach the cached Bearer token via an HTTP client middleware or Symfony scoped client default header.
```

**[Stale Cache Fallback](https://michalsniezko.github.io/microservices-observability/stale-cache-fallback.html)**
```
For resilience against upstream failures: cache responses without a hard TTL. Track expiry manually via cached_at. On cache hit + expired: attempt the upstream call; if it fails, serve the stale entry. Always log when serving stale data so degradation is observable.
```

---

### AWS & Infrastructure

**[SNS Fan-out](https://michalsniezko.github.io/aws-event-driven/sns-fanout.html)**
```
For multi-consumer event delivery: publish once to an SNS topic. Each consumer owns its SQS queue and subscribes independently. Publishers have no knowledge of consumers. Attach a DLQ to each SQS subscription independently.
```

**[SNS Filter Policies](https://michalsniezko.github.io/aws-event-driven/sns-filters.html)**
```
Route messages selectively by attaching filter policies to SNS subscriptions, not to the publisher. Use attribute-based filtering for simple key-value matching. Set FilterPolicyScope: MessageBody to filter on body content. Consumers receive only messages they opted into.
```

**[Out-of-Order Message Handling](https://michalsniezko.github.io/aws-event-driven/out-of-order.html)**
```
For SQS consumers with stateful entities: store event_timestamp on every message. At the consumer, compare against entity's last_processed_at. Discard the message if it is older than the last-applied event. Update last_processed_at only when applying a newer event.
```

**[Adaptive Message Splitting](https://michalsniezko.github.io/aws-event-driven/adaptive-message-splitting.html)**
```
For payloads that may exceed SNS's 256KB limit: use a generator-based strategy that adds items one at a time and flushes a chunk when the serialized size would exceed the limit. Do not pre-compute sizes. Check after each addition and flush before the limit, not after.
```

**[SQS-Based Autoscaling](https://michalsniezko.github.io/scaling-cloudwatch-autoscaling/sqs-based-autoscaling.html)**
```
Scale ECS workers on BacklogPerInstance = ApproximateNumberOfMessagesVisible / running task count, not CPU. Publish this as a custom CloudWatch metric from a scheduled Lambda. Set desired task count to ceil(queue_depth / target_messages_per_instance).
```

**[AWS SDK Pitfalls](https://michalsniezko.github.io/scaling-cloudwatch-autoscaling/aws-sdk-pitfalls.html)**
```
Initialize AWS SDK clients (SQS, SNS, S3, DynamoDB) once at module level, outside the Lambda handler function. Reuse across warm invocations. Never instantiate SDK clients inside the handler body - it adds cold-start latency and can exhaust connection pools under concurrency.
```

**[Terraform Variables and Jenkins](https://michalsniezko.github.io/devops-infrastructure-cicd/terraform-required-variable-jenkins.html)**
```
Every Terraform variable must have a default value unless interactive input at plan time is intentional. A variable without a default blocks unattended Jenkins runs waiting for a prompt that never arrives. Use description to document the variable's purpose, especially if it is not directly wired to a resource.
```

**[Terraform Project Structure](https://michalsniezko.github.io/devops-infrastructure-cicd/terraform-project-structure.html)**
```
Organize Terraform by environment directory. Store state in S3 with DynamoDB locking. Always plan to a binary (terraform plan -out=plan.bin) and apply the binary. Never run terraform apply without a reviewed plan - drift can occur between plan and apply otherwise.
```

**[SSM Parameter Store](https://michalsniezko.github.io/devops-infrastructure-cicd/aws-ssm-parameter-store.html)**
```
Store all secrets in SSM Parameter Store as SecureString. Reference by ARN in ECS task definitions or Terraform locals. Never commit secrets to Git. Never pass secrets as plain environment variables in docker-compose or CI config files.
```

---

### Frontend & JavaScript

**[Effector](https://michalsniezko.github.io/js-frontend-tooling/effector.html)**
```
Effector conventions: prefix stores with $, suffix effects with Fx. Use .on(event, reducer) for simple store updates. Use sample({ clock, source, filter, fn, target }) for declarative unit wiring. Use createEffect for all async work - .pending, .doneData, .failData are available for free. Test with fork({ handlers }) and allSettled() for isolated scopes without React rendering. Never mutate store state directly.
```

**[Rollup / esbuild for Libraries](https://michalsniezko.github.io/js-frontend-tooling/rollup-bundler.html)**
```
When building a shared JS/TS library: list everything in peerDependencies in the bundler's external[] array or you will bundle duplicates of React/axios into consumers. Ship both ESM (dist/index.esm.js) and CJS (dist/index.cjs.js). For new projects prefer esbuild - it is 10-100x faster than Rollup with built-in TypeScript support.
```

**[Wrap Interceptor (Lambda)](https://michalsniezko.github.io/js-frontend-tooling/wrap-interceptor-pattern.html)**
```
For Node.js Lambda handlers: wrap business logic with a wrapHandler() higher-order function that handles request parsing, auth checks, structured logging, error formatting, and response shaping. Business logic receives parsed input and returns data. Keep the interceptor to 3 concerns max.
```

---

### Observability

**[Kibana Log Browsing](https://michalsniezko.github.io/monitoring-js-tooling/kibana-log-browsing.html)**
```
When debugging with Kibana: filter by extra.traceId: "<id>" to see the full cross-service request journey. Use absolute time ranges during incidents. Add service, level, log_message, extra.traceId as Discover columns. Save frequent queries. Use View surrounding documents when you lack a trace ID. Switch to Lucene syntax for regex queries on log messages.
```

**[ELK Stack](https://michalsniezko.github.io/monitoring-js-tooling/elk-stack.html)**
```
Log all service output as structured JSON with trace_id, service, level, and duration_ms fields. Use per-service, per-day index patterns (logs-service-YYYY.MM.dd) for targeted retention policies. Set number_of_replicas: 0 on single-node dev setups to avoid yellow index health warnings. Configure ILM policies to prevent disk exhaustion.
```
