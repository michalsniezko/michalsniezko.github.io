---
layout: default
title: "Routing Logic: endpoints.yaml vs routes.yaml"
parent: Microservices & Service Design
nav_order: 6
---

## Routing Logic: `endpoints.yaml` vs `routes.yaml`

**Context:** In a microservice, you deal with two directions of traffic: **outgoing** calls to other services and **incoming** requests to your own endpoints. Mixing these configs leads to confusion about what your service exposes vs. what it consumes.

### `routes.yaml` - Incoming Traffic (What You Expose)

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

### `endpoints.yaml` - Outgoing Traffic (What You Consume)

Defines base URLs and paths for services you call. Typically loaded into service client configuration.

```yaml
# config/endpoints.yaml
services:
    inventory_service:
        base_url: "http://inventory-service.svc:8080"
        endpoints:
            get_product: "/api/v1/products/{productId}"
            search_products: "/api/v1/products"

    pricing_service:
        base_url: "http://pricing-service.svc:8080"
        endpoints:
            calculate_price: "/api/v1/prices/calculate"
```

### Wiring Endpoints into a Service Client

```yaml
# services.yaml
App\Client\InventoryClient:
    arguments:
        $baseUrl: '%inventory_service.base_url%'
        $endpoints: '%inventory_service.endpoints%'
```

```php
class InventoryClient
{
    public function __construct(
        private HttpClientInterface $httpClient,
        private string $baseUrl,
        private array $endpoints
    ) {}

    public function getProduct(string $productId): array
    {
        $path = str_replace('{productId}', $productId, $this->endpoints['get_product']);

        return $this->httpClient->request('GET', $this->baseUrl . $path)->toArray();
    }
}
```

> **Gotcha:** When a team renames their API path (e.g., `/api/v1/products` → `/api/v2/products`), you only need to update `endpoints.yaml` - not grep through business logic. But this only works if you never hardcode paths in PHP code. The moment someone writes `$client->get('http://inventory-service.svc:8080/api/v1/products/' . $id)` inline, you've lost the single source of truth. Enforce that all outgoing paths come from config.

---

### For AI agents

```
Separate outgoing service endpoints (endpoints.yaml) from incoming routes (routes.yaml). Incoming contracts and outgoing dependencies must not be defined in the same file. This makes it clear which config belongs to your service's public API vs. its external dependencies.
```

Reference: `https://michalsniezko.github.io/microservices-observability/routing-endpoints-routes.html`
