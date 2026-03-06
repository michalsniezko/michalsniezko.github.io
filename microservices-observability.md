# Microservices: Service Discovery & Observability

## Repositories as Service Clients

**Context:** In a monolith, a `Repository` talks to a local database. In a microservices architecture, the same pattern applies — but the "data source" is another service's API. The repository abstraction hides whether data comes from Postgres or an HTTP call, keeping your domain layer clean.

### Example: A Repository That Calls an External Service

```php
class VehicleRepository
{
    private HttpClientInterface $client;
    private string $baseUrl;

    public function __construct(HttpClientInterface $client, string $baseUrl)
    {
        $this->client = $client;
        $this->baseUrl = $baseUrl;
    }

    public function findById(string $vehicleId): ?VehicleDTO
    {
        $response = $this->client->request('GET', sprintf(
            '%s/api/v1/vehicles/%s',
            $this->baseUrl,
            $vehicleId
        ));

        if ($response->getStatusCode() === 404) {
            return null;
        }

        return VehicleDTO::fromArray($response->toArray());
    }
}
```

### Service Definition (Symfony)

```yaml
# services.yaml
App\Repository\VehicleRepository:
    arguments:
        $baseUrl: '%env(VEHICLE_SERVICE_URL)%'
```

> **Gotcha:** Treat HTTP repositories as unreliable data sources. Unlike a DB query, a service call can timeout, return 503, or give you stale data. Always define explicit timeouts on the HTTP client and decide on a fallback strategy (throw, return null, use cache) — don't let Guzzle's default 30s timeout silently stall your request.

---

## Consul & `.svc` Endpoints

**Context:** Hardcoding service URLs (`http://10.0.3.47:8080`) breaks the moment an instance scales, moves, or dies. A service mesh like Consul maintains a registry of healthy instances and provides stable DNS names so services find each other without caring about infrastructure.

### How It Works

<pre class="mermaid">
graph TD
    subgraph Client
    A[Billing Service]
    end

    subgraph Discovery
    B[Consul DNS]
    end

    subgraph "Vehicle Service Cluster"
    I1[Instance-1 <br/>10.0.3.47 <br/><b>Healthy</b>]
    I2[Instance-2 <br/>10.0.3.48 <br/><b>Healthy</b>]
    I3[Instance-3 <br/>10.0.3.49 <br/><b>Failing ✗</b>]
    end

    A -- "DNS query: vehicle-service.svc" --> B
    B -- "A record: 10.0.3.47" --> A
    
    B -. checks .- I1
    B -. checks .- I2
    B -. checks .- I3

    style I1 fill:#d4edda,stroke:#28a745
    style I2 fill:#d4edda,stroke:#28a745
    style I3 fill:#f8d7da,stroke:#dc3545
</pre>

- Services register with Consul on startup and send periodic health checks.
- Consul removes unhealthy instances from DNS responses.
- Consumers use internal `.svc` DNS names — no IPs, no load balancer config changes needed.

### Environment Config

```yaml
# endpoints.yaml (outgoing calls)
vehicle_service:
    base_url: "http://vehicle-service.svc:8080"

pricing_service:
    base_url: "http://pricing-service.svc:8080"
```

### Consul Health Check (Registered by the Service)

```json
{
  "service": {
    "name": "vehicle-service",
    "port": 8080,
    "check": {
      "http": "http://localhost:8080/health",
      "interval": "10s",
      "timeout": "2s",
      "deregister_critical_service_after": "30s"
    }
  }
}
```

> **Gotcha:** If your health check endpoint hits the database or a downstream dependency, a DB outage will mark your service as unhealthy — even though the service process itself is fine. Keep `/health` lightweight (return 200 if the process is up). Use a separate `/health/ready` for deep checks that include dependencies.

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

---

## Monolog Integration with Zipkin Processor

**Context:** Tracing headers exist on the HTTP layer, but your application logs (Monolog) don't automatically include them. Without adding the TraceID to every log entry, you can't correlate application-level logs with distributed traces.

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

> **Gotcha:** If you run console commands or async workers (Messenger), there's no incoming HTTP request — `RequestStack::getCurrentRequest()` returns `null`. Your processor must handle this gracefully (like the `'no-trace'` fallback above), or better yet, generate a synthetic trace ID for CLI contexts so those logs are also traceable.

---

## API Documentation with Swagger/OpenAPI Annotations

**Context:** Maintaining a separate API doc that drifts from the actual code is a liability. Annotating endpoints directly in the controller keeps documentation co-located with the implementation — when the code changes, the doc reminder is right there.

### Annotated Controller (PHP with OpenAPI Attributes)

```php
use OpenApi\Attributes as OA;

class OrderController
{
    #[OA\Get(
        path: '/api/v1/orders/{orderId}',
        summary: 'Retrieve a single order by ID',
        tags: ['Orders'],
        parameters: [
            new OA\Parameter(
                name: 'orderId',
                in: 'path',
                required: true,
                schema: new OA\Schema(type: 'string', format: 'uuid')
            )
        ],
        responses: [
            new OA\Response(
                response: 200,
                description: 'Order details',
                content: new OA\JsonContent(ref: '#/components/schemas/OrderResponse')
            ),
            new OA\Response(response: 404, description: 'Order not found')
        ]
    )]
    public function getOrder(string $orderId): JsonResponse
    {
        // ...
    }
}
```

### Generating the Spec

```bash
# Generate OpenAPI JSON from annotations
./vendor/bin/openapi src/ --output public/api-docs/openapi.json

# Serve with Swagger UI (Docker, for local dev)
docker run -p 8082:8080 \
  -e SWAGGER_JSON=/spec/openapi.json \
  -v $(pwd)/public/api-docs:/spec \
  swaggerapi/swagger-ui
```

> **Gotcha:** Annotations only document what you write — they don't validate runtime behavior. If your annotation says the response is `200` with an `OrderResponse` schema but your code actually returns a different structure on edge cases, the doc becomes a lie. Pair annotations with response serialization tests that assert the actual output matches the documented schema.

---

## Routing Logic: `endpoints.yaml` vs `routes.yaml`

**Context:** In a microservice, you deal with two directions of traffic: **outgoing** calls to other services and **incoming** requests to your own endpoints. Mixing these configs leads to confusion about what your service exposes vs. what it consumes.

### `routes.yaml` — Incoming Traffic (What You Expose)

Defines the HTTP endpoints your service handles. This is your service's public contract.

```yaml
# config/routes.yaml
order_get:
    path: /api/v1/orders/{orderId}
    controller: App\Controller\OrderController::getOrder
    methods: [GET]

order_create:
    path: /api/v1/orders
    controller: App\Controller\OrderController::createOrder
    methods: [POST]

health_check:
    path: /health
    controller: App\Controller\HealthController::check
    methods: [GET]
```

### `endpoints.yaml` — Outgoing Traffic (What You Consume)

Defines base URLs and paths for services you call. Typically loaded into service client configuration.

```yaml
# config/endpoints.yaml
services:
    vehicle_service:
        base_url: "http://vehicle-service.svc:8080"
        endpoints:
            get_vehicle: "/api/v1/vehicles/{vehicleId}"
            search_vehicles: "/api/v1/vehicles"

    pricing_service:
        base_url: "http://pricing-service.svc:8080"
        endpoints:
            calculate_price: "/api/v1/prices/calculate"
```

### Wiring Endpoints into a Service Client

```yaml
# services.yaml
App\Client\VehicleClient:
    arguments:
        $baseUrl: '%vehicle_service.base_url%'
        $endpoints: '%vehicle_service.endpoints%'
```

```php
class VehicleClient
{
    public function __construct(
        private HttpClientInterface $httpClient,
        private string $baseUrl,
        private array $endpoints
    ) {}

    public function getVehicle(string $vehicleId): array
    {
        $path = str_replace('{vehicleId}', $vehicleId, $this->endpoints['get_vehicle']);

        return $this->httpClient->request('GET', $this->baseUrl . $path)->toArray();
    }
}
```

> **Gotcha:** When a team renames their API path (e.g., `/api/v1/vehicles` → `/api/v2/vehicles`), you only need to update `endpoints.yaml` — not grep through business logic. But this only works if you never hardcode paths in PHP code. The moment someone writes `$client->get('http://vehicle-service.svc:8080/api/v1/vehicles/' . $id)` inline, you've lost the single source of truth. Enforce that all outgoing paths come from config.

<script type="module">
	import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
	mermaid.initialize({
		startOnLoad: true,
		theme: 'dark'
	});
</script>
