---
layout: default
title: Repositories as Service Clients
parent: Microservices
nav_order: 1
---

## Repositories as Service Clients

**Context:** In a monolith, a `Repository` talks to a local database. In a microservices architecture, the same pattern applies - but the "data source" is another service's API. The repository abstraction hides whether data comes from Postgres or an HTTP call, keeping your domain layer clean.

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

> **Gotcha:** Treat HTTP repositories as unreliable data sources. Unlike a DB query, a service call can timeout, return 503, or give you stale data. Always define explicit timeouts on the HTTP client and decide on a fallback strategy (throw, return null, use cache) - don't let Guzzle's default 30s timeout silently stall your request.
