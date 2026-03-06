---
layout: default
title: WireMock for External API Simulation
parent: Testing, Concurrency, Distributed Locks
nav_order: 4
---

## WireMock for External API Simulation

**Scenario:** Your service calls a payment provider. You need to test: what happens when the provider responds in 5 seconds? What about a `502`? What about valid JSON with an unexpected field? You can't (and shouldn't) trigger these conditions on the real provider during tests.

### Simulating Slow Responses

```json
{
  "request": {
    "method": "POST",
    "urlPath": "/api/v1/payments/charge"
  },
  "response": {
    "status": 200,
    "fixedDelayMilliseconds": 5000,
    "jsonBody": {
      "transactionId": "txn-slow-123",
      "status": "completed"
    }
  }
}
```

### Simulating Intermittent Failures (Fault Injection)

```json
{
  "request": {
    "method": "POST",
    "urlPath": "/api/v1/payments/charge"
  },
  "response": {
    "fault": "CONNECTION_RESET_BY_PEER"
  }
}
```

### Available WireMock Faults

| Fault                         | Simulates                                  |
|-------------------------------|--------------------------------------------|
| `CONNECTION_RESET_BY_PEER`    | TCP connection dropped mid-request         |
| `EMPTY_RESPONSE`              | Server accepts connection, sends nothing   |
| `MALFORMED_RESPONSE_CHUNK`    | Garbage bytes in the HTTP response         |
| `RANDOM_DATA_THEN_CLOSE`      | Random data, then connection close         |

### PHP Integration Test with Timeout Assertion

```php
class PaymentClientTimeoutTest extends TestCase
{
    public function testClientTimesOutOnSlowResponse(): void
    {
        // WireMock stub returns after 5s, client timeout is 3s
        $this->registerWireMockStub([
            'request'  => ['method' => 'POST', 'urlPath' => '/api/v1/payments/charge'],
            'response' => [
                'status' => 200,
                'fixedDelayMilliseconds' => 5000,
                'jsonBody' => ['transactionId' => 'txn-slow'],
            ],
        ]);

        $client = new PaymentClient(
            HttpClient::create(['timeout' => 3]),
            'http://localhost:8080'
        );

        $this->expectException(TransportExceptionInterface::class);
        $client->charge('ord-123', 49.99);
    }

    public function testClientRetriesOnConnectionReset(): void
    {
        // First call: connection reset. Second call: success.
        $this->registerWireMockStub([
            'request'       => ['method' => 'POST', 'urlPath' => '/api/v1/payments/charge'],
            'response'      => ['fault' => 'CONNECTION_RESET_BY_PEER'],
            'scenarioName'  => 'retry-scenario',
            'requiredScenarioState' => 'Started',
            'newScenarioState'      => 'After-Failure',
        ]);

        $this->registerWireMockStub([
            'request'       => ['method' => 'POST', 'urlPath' => '/api/v1/payments/charge'],
            'response'      => ['status' => 200, 'jsonBody' => ['transactionId' => 'txn-retry-ok']],
            'scenarioName'  => 'retry-scenario',
            'requiredScenarioState' => 'After-Failure',
        ]);

        $client = new PaymentClient(
            HttpClient::create(),
            'http://localhost:8080'
        );

        $result = $client->chargeWithRetry('ord-123', 49.99, maxRetries: 2);
        self::assertSame('txn-retry-ok', $result->transactionId);
    }
}
```

> **Safety First:** When testing timeouts, set WireMock's delay *higher* than your client timeout — otherwise the test passes because the response arrives in time, not because your timeout handling works. Also: WireMock scenarios are global state. If your test suite runs in parallel (`paratest`), use unique scenario names per test class or reset WireMock state between test classes.
