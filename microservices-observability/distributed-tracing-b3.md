---
layout: default
title: Distributed Tracing with X-B3 Headers
parent: Microservices
nav_order: 3
---

## Distributed Tracing with X-B3 Headers (Zipkin)

**Context:** A single user request might pass through an API gateway, auth service, order service, payment service, and notification service. Without a shared trace ID, correlating logs across these systems is guesswork. Zipkin's B3 propagation solves this by threading a `TraceID` through every hop.

### B3 Header Set

| Header                  | Purpose                                    | Example                            |
|-------------------------|--------------------------------------------|------------------------------------|
| `X-B3-TraceId`          | Global ID for the entire request chain     | `463ac35c9f6413ad48485a3953bb6124` |
| `X-B3-SpanId`           | ID for the current operation               | `0020000000000001`                 |
| `X-B3-ParentSpanId`     | SpanID of the caller                       | `0010000000000001`                 |
| `X-B3-Sampled`          | `1` = trace this request, `0` = skip       | `1`                                |

### Propagation Flow

```
User ──► API Gateway ──► Order Service ──► Payment Service
         TraceId: aaa      TraceId: aaa      TraceId: aaa
         SpanId:  001      SpanId:  002      SpanId:  003
                           ParentSpan: 001   ParentSpan: 002
```

The `TraceId` stays the same. Each service creates a new `SpanId` and sets `ParentSpanId` to the caller's span.

### Forwarding B3 Headers in a Guzzle Client (PHP)

```php
class TracingMiddleware
{
    private const B3_HEADERS = [
        'X-B3-TraceId',
        'X-B3-SpanId',
        'X-B3-ParentSpanId',
        'X-B3-Sampled',
    ];

    public static function forward(RequestInterface $currentRequest): callable
    {
        return static function (callable $handler) use ($currentRequest) {
            return static function ($request, array $options) use ($handler, $currentRequest) {
                foreach (self::B3_HEADERS as $header) {
                    $value = $currentRequest->headers->get($header);
                    if ($value !== null) {
                        $request = $request->withHeader($header, $value);
                    }
                }

                return $handler($request, $options);
            };
        };
    }
}
```

### Viewing in Kibana

With the TraceID indexed in your logs, a Kibana query like:

```
traceId: "463ac35c9f6413ad48485a3953bb6124"
```

...returns every log line from every service that touched that request, in chronological order.

> **Gotcha:** The most common tracing break: making an outgoing HTTP call without forwarding B3 headers. If even one service in the chain drops them, the downstream services generate new trace IDs and the request becomes invisible in your trace view. Audit every HTTP client in your codebase — Guzzle, cURL, Symfony HttpClient — to ensure middleware or interceptors propagate headers.
