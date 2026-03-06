---
layout: default
title: WireMock for Endpoint Testing
parent: Backend Patterns & Optimization for High-Volume Systems
nav_order: 4
---

## WireMock for Endpoint Testing

**Problem:** Integration tests that hit real external services are slow, flaky, and dependent on third-party uptime. Mocking at the PHP level (Prophecy/PHPUnit) skips the HTTP layer entirely - you miss serialization bugs, wrong headers, and real HTTP status handling.

**Solution:** WireMock runs a real HTTP server that returns canned responses. Your application makes actual HTTP calls to `localhost:8080` (WireMock), so the full HTTP client stack is exercised.

### WireMock Stub Definition

```json
{
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/v1/vehicles/[a-f0-9-]+",
    "headers": {
      "Accept": { "equalTo": "application/json" }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": {
      "id": "abc-123",
      "make": "Toyota",
      "model": "Corolla",
      "year": 2024
    }
  }
}
```

### Docker Compose Setup

```yaml
# docker-compose.test.yml
services:
  wiremock:
    image: wiremock/wiremock:3.5.4
    ports:
      - "8080:8080"
    volumes:
      - ./tests/wiremock/mappings:/home/wiremock/mappings
      - ./tests/wiremock/__files:/home/wiremock/__files
    command: --verbose
```

### Test Using WireMock

```php
class VehicleClientIntegrationTest extends TestCase
{
    private VehicleClient $client;

    protected function setUp(): void
    {
        $this->client = new VehicleClient(
            HttpClient::create(),
            'http://localhost:8080' // WireMock
        );
    }

    public function testGetVehicleReturnsDTO(): void
    {
        // WireMock mapping already loaded from ./tests/wiremock/mappings/
        $vehicle = $this->client->getVehicle('abc-123');

        self::assertSame('Toyota', $vehicle->make);
        self::assertSame(2024, $vehicle->year);
    }

    public function testGetVehicleHandles500(): void
    {
        // Use WireMock admin API to register fault scenario
        $this->registerWireMockStub([
            'request' => ['method' => 'GET', 'urlPath' => '/api/v1/vehicles/fail-id'],
            'response' => ['status' => 500, 'body' => 'Internal Server Error'],
        ]);

        $this->expectException(UpstreamServiceException::class);
        $this->client->getVehicle('fail-id');
    }

    private function registerWireMockStub(array $mapping): void
    {
        HttpClient::create()->request('POST', 'http://localhost:8080/__admin/mappings', [
            'json' => $mapping,
        ]);
    }
}
```

> **Gotcha:** WireMock stubs are stateful per server instance. If tests run in parallel, one test's stub can leak into another. Use `"scenarioName"` for stateful sequences, and call `POST /__admin/reset` in `setUp()` if tests share a WireMock instance. Or use `docker compose` to spin up an isolated instance per test suite.

