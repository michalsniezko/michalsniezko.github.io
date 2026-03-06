---
layout: default
title: Monolog Integration with Zipkin Processor
parent: Microservices
nav_order: 4
---

## Monolog Integration with Zipkin Processor

**Context:** Tracing headers exist on the HTTP layer, but your application logs ([Monolog](https://github.com/Seldaek/monolog)) don't automatically include them. Without adding the TraceID to every log entry, you can't correlate application-level logs with distributed traces.

### Custom Monolog Processor

```php
class ZipkinProcessor
{
    private RequestStack $requestStack;

    public function __construct(RequestStack $requestStack)
    {
        $this->requestStack = $requestStack;
    }

    public function __invoke(array $record): array
    {
        $request = $this->requestStack->getCurrentRequest();

        if ($request === null) {
            return $record;
        }

        $record['extra']['traceId'] = $request->headers->get('X-B3-TraceId', 'no-trace');
        $record['extra']['spanId']  = $request->headers->get('X-B3-SpanId', 'no-span');

        return $record;
    }
}
```

### Monolog Config (Symfony)

```yaml
# config/packages/monolog.yaml
monolog:
    handlers:
        main:
            type: stream
            path: "%kernel.logs_dir%/%kernel.environment%.log"
            level: info
            formatter: monolog.formatter.json
    processors:
        - App\Logging\ZipkinProcessor
```

### Resulting Log Output

```json
{
  "message": "Order created successfully",
  "channel": "app",
  "level_name": "INFO",
  "datetime": "2026-03-06T10:15:30+00:00",
  "extra": {
    "traceId": "463ac35c9f6413ad48485a3953bb6124",
    "spanId": "0020000000000001"
  }
}
```

> **Gotcha:** If you run console commands or async workers (Messenger), there's no incoming HTTP request - `RequestStack::getCurrentRequest()` returns `null`. Your processor must handle this gracefully (like the `'no-trace'` fallback above), or better yet, generate a synthetic trace ID for CLI contexts so those logs are also traceable.
