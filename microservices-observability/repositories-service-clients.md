---
layout: default
title: Repositories as Service Clients
parent: Microservices & Service Design
nav_order: 1
---

## Repositories as Service Clients

**Context:** In a monolith, a `Repository` talks to a local database. In a microservices architecture, the same pattern applies - but the "data source" is another service's API. The repository abstraction hides whether data comes from Postgres or an HTTP call, keeping your domain layer clean.

### Example: A Repository That Calls an External Service

Configure the HTTP client as a [scoped client](https://symfony.com/doc/current/http_client.html#scoping-client) in `framework.yaml`. This keeps the base URL and default headers out of PHP code entirely, and lets Symfony inject a pre-configured client directly:

```yaml
# config/packages/framework.yaml
framework:
    http_client:
        scoped_clients:
            vehicle_service.client:
                base_uri: '%env(VEHICLE_SERVICE_URL)%'
                headers:
                    Accept: 'application/json'
```

The repository receives the ready-to-use client and works with relative URLs only:

```php
class VehicleRepository
{
    public function __construct(private HttpClientInterface $client) {}

    public function findById(string $vehicleId): ?VehicleDTO
    {
        $response = $this->client->request(
            'GET',
            '/api/v1/vehicles/' . $vehicleId
        );

        if ($response->getStatusCode() === 404) {
            return null;
        }

        return VehicleDTO::fromArray($response->toArray());
    }
}
```

```yaml
# services.yaml - Symfony autowires by name when the argument matches the scoped client
App\Repository\VehicleRepository:
    arguments:
        $client: '@vehicle_service.client'
```

The repository no longer knows or cares about the base URL. Changing the upstream service URL is a config change, not a code change.

> **Gotcha:** Treat HTTP repositories as unreliable data sources. Unlike a DB query, a service call can timeout, return 503, or give you stale data. Always define explicit timeouts on the HTTP client and decide on a fallback strategy (throw, return null, use cache) - don't let Guzzle's default 30s timeout silently stall your request.

For service-to-service security, clients like this need an OAuth2 Bearer token on every request; the [OAuth2 in a Microservice Environment](oauth2.md) article covers the Client Credentials flow and token caching. For resilience when the upstream is temporarily unavailable, the [Stale Cache Fallback](stale-cache-fallback.md) pattern lets the repository serve a cached response rather than propagating the failure.

---

### For AI agents

```
For HTTP calls to upstream services: configure a Symfony scoped client with base_uri in framework.yaml. Inject the pre-configured client into the repository by name. Use relative URLs in the repository. Never inject base URLs as constructor string arguments.
```

Reference: `https://michalsniezko.github.io/microservices-observability/repositories-service-clients.html`
