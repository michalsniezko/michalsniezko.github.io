---
layout: default
title: Kibana Log Browsing
parent: Self-Hosted Monitoring
nav_order: 5
---

## Kibana Log Browsing

Kibana's Discover view is the primary tool for debugging live and historical issues. This article covers how to navigate logs effectively, trace requests across services using trace IDs, and avoid the patterns that waste time during an incident.

For the ELK stack setup and how logs get from your services into Elasticsearch, see [The ELK Stack](elk-stack.md). For how trace IDs are generated and propagated across services, see [Distributed Tracing with X-B3 Headers](../microservices-observability/distributed-tracing-b3.md) and [Monolog Zipkin Processor](../microservices-observability/monolog-zipkin-processor.md).

---

### KQL: Kibana Query Language

KQL is the query syntax used in the Discover search bar. It is not Elasticsearch's full query DSL - it's a simplified layer on top designed for log browsing.

**Field match:**
```
level: error
```

**Multiple values for the same field (OR):**
```
level: (error or warn)
```

**AND conditions:**
```
level: error AND service: payment-service
```

**Wildcard on a field value:**
```
log_message: "Payment*"
```

**Range queries:**
```
duration_ms > 1000
duration_ms >= 500 AND duration_ms <= 2000
```

**Nested field:**
```
context.order_id: "ord-4821"
```

**Existence check — field is present:**
```
context.exception: *
```

**Negation:**
```
NOT level: debug
service: order-service AND NOT context.status_code: 200
```

**Free-text search across all fields:**
```
"Card declined"
```

Use quotes for exact phrases. Without quotes, each word is searched independently and you may get false matches.

---

### Tracing a Request with a Trace ID

When a request fails, the fastest path to the root cause is filtering all logs by the trace ID of that request. If your services propagate B3 headers and include `trace_id` in every log line (as described in [Monolog Zipkin Processor](../microservices-observability/monolog-zipkin-processor.md)), a single query surfaces the full request journey across every service that touched it.

```
extra.traceId: "463ac35c9f6413ad48485a3953bb6124"
```

What you get back is every log line — from every service — that belongs to that trace, in chronological order. You can see exactly where the error originated, what the upstream caller received, and what downstream services were triggered.

**Workflow for debugging a reported error:**

1. Get the trace ID from the error report, alert, or from the user's request headers
2. Set the time range to cover when the error happened (plus a few minutes either side)
3. Query `extra.traceId: "<id>"`
4. Sort by `@timestamp` ascending to read the request flow in order
5. Look for the first `level: error` or unexpected status code

If you don't have the trace ID, start from the symptom instead:

```
level: error AND service: payment-service AND context.order_id: "ord-4821"
```

Once you find a relevant log line, the trace ID is in its `extra.traceId` field. Copy it and re-query to see the full picture.

---

### The Time Range Selector

The time picker in the top right is one of the most important controls. Bad time ranges are responsible for more wasted debugging time than bad queries.

**Default is "Last 15 minutes"** - this is almost always too short for incident investigation. Change it to "Last 1 hour" or set an absolute range around when the incident occurred.

**Absolute ranges are more reliable during incidents.** If you know the error happened around 14:30, set `2026-04-01 14:20` to `2026-04-01 14:45` rather than a relative range that keeps shifting as you browse.

**Refresh rate:** For live tail during an active incident, enable auto-refresh (the clock icon next to the time picker). 5-10 seconds is enough for most log volumes without hammering Elasticsearch.

---

### Columns and the Document Table

By default, Kibana shows `@timestamp` and the raw `_source` JSON. This is noisy. Add the columns you actually care about:

1. In Discover, click the field name in the left panel to expand it
2. Click the `+` icon next to a field to add it as a column
3. Useful columns to always have: `service`, `level`, `log_message`, `extra.traceId`, `duration_ms`, `context.status_code`

The column configuration persists per saved search, so set it up once and save it.

**Expanding a document:** Click the `>` arrow on any row to expand the full document. This shows every field in the log entry. Useful when you need to inspect context fields that aren't in your column set.

---

### Saved Searches

Save frequent queries as named searches. Examples worth saving:

| Name | Query |
|---|---|
| All errors | `level: error` |
| Slow requests | `duration_ms > 1000` |
| Payment errors | `level: error AND service: payment-service` |
| Unhandled exceptions | `context.exception: *` |
| 5xx responses | `context.status_code >= 500` |

Access saved searches from the Discover menu or link to them directly in incident runbooks.

---

### Filtering by Clicking

In Discover, clicking on any value in the document table offers instant filter options:

- **Filter for value** - adds a positive filter (equivalent to `field: value` in KQL)
- **Filter out value** - adds a negative filter (equivalent to `NOT field: value`)

This is faster than typing for interactive exploration. Build up filters by clicking through log lines rather than writing KQL from scratch. The active filters appear as chips below the search bar and can be toggled off without retyping.

---

### Surrounding Documents

When you find a relevant log entry, you can view the logs immediately before and after it in time — across all services — without knowing the trace ID. In the expanded document view, click **View surrounding documents**.

This is useful when:
- The problematic event doesn't have a trace ID (CLI jobs, async workers)
- You want to see what was happening system-wide just before an error
- You're investigating a cascading failure and want to see which service degraded first

---

### Using Lucene Syntax for Advanced Queries

KQL covers most cases, but switching to Lucene syntax (toggle in the search bar) unlocks regex and fuzzy matching:

**Regex on a field:**
```
log_message:/Payment (failed|declined|rejected)/
```

**Fuzzy match (useful for typos in log messages):**
```
log_message:excepiton~
```

**Wildcard in the middle:**
```
context.order_id: ord-48*
```

Note: regex on high-cardinality fields (like `log_message`) is expensive. Use it narrowed to a small time range.

---

### Dashboards for Recurring Patterns

Discover is for ad-hoc investigation. For recurring questions, build a dashboard in Kibana's Dashboard view:

**Error rate over time:**
- Visualisation type: Bar chart or line chart
- Y-axis: Count of documents
- Filter: `level: error`
- Break down by: `service` (using a Terms aggregation)

**p99 response time per service:**
- Visualisation type: Line chart
- Y-axis: 99th percentile of `duration_ms`
- Break down by: `service`

**Top error messages:**
- Visualisation type: Data table
- Bucket: Terms on `log_message.keyword`
- Filter: `level: error`

**5xx rate:**
- Visualisation type: Metric or gauge
- Filter: `context.status_code >= 500`
- Compare to total request count with a secondary metric

Dashboards are most useful when pinned to a TV during incidents or checked at the start of each working day as a health check.

---

### Index Pattern and Field Mapping Gotchas

**`.keyword` suffix:** Elasticsearch maps `text` fields for full-text search and `keyword` fields for exact match and aggregations. If your field is mapped as `text`, you can search it but can't aggregate it. For aggregations (dashboards, top-N), use `field.keyword`. For search queries, use `field` directly.

**Date format issues:** If `@timestamp` isn't being parsed as a date, Kibana can't use the time picker to filter by it. Check the index mapping: `GET /logs-*/_mapping` should show `@timestamp` as `date` type. If it's `text`, the Logstash or Filebeat pipeline isn't setting it correctly.

**Missing fields in the left panel:** Kibana only shows fields that appear in at least one document in the current time range. If a field is missing from the field list, either no documents in the current range have it, or the index pattern's field list is stale. Refresh it under Stack Management > Index Patterns > Refresh field list.

---

### Useful Keyboard Shortcuts

| Action | Shortcut |
|---|---|
| Focus search bar | Click or `/` |
| Run query | `Enter` |
| Expand time range backwards | Shift+click the histogram bar to the left |
| Open a document | Click `>` on the row |
| Add column from expanded doc | Click field name → `Toggle column` |

---

### What to Do When Kibana is Slow

Slow Kibana queries are almost always caused by one of three things:

**Time range too wide.** A 30-day search across millions of documents is slow. Narrow to the relevant window first.

**Wildcard at the start of a term.** `log_message: *error*` forces a full index scan. Prefer prefix wildcards or exact matches where possible.

**High-cardinality Terms aggregation.** Aggregating on a field with millions of unique values (like `context.order_id`) consumes significant heap. Use filter aggregations to narrow the set first, then aggregate.

If Elasticsearch itself is under pressure, check the cluster health via `GET /_cluster/health` and monitor heap usage in Stack Monitoring. Shard count and replica configuration have a large impact on query performance — see the [ELK Stack](elk-stack.md) article for ILM and shard sizing guidance.
