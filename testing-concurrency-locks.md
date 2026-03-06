---
layout: default
title: Testing, Concurrency, Distributed Locks
nav_order: 5
---

# Testing, Concurrency & Distributed Locks

## Distributed Lock (Symfony Lock)

**Scenario:** Two Kubernetes pods receive the same SQS message (at-least-once delivery). Both try to generate an invoice for order `ord-999`. Without coordination, you get duplicate invoices and a very confused accounting team.

A distributed lock ensures only one process executes a critical section at a time, even across multiple nodes. The lock state lives in a shared store (Redis, PostgreSQL, DynamoDB) — not in local memory.

### Blocking vs. Non-Blocking

| Strategy         | Behavior                                    | Use When                                |
|------------------|---------------------------------------------|-----------------------------------------|
| **Blocking**     | Waits (up to TTL) for the lock to release   | Work must eventually complete — e.g., invoice generation |
| **Non-Blocking** | Returns `false` immediately if lock is held | Duplicate work is safe to skip — e.g., idempotent cache refresh |

### Blocking Lock

```php
use Symfony\Component\Lock\LockFactory;

class InvoiceGenerator
{
    public function __construct(
        private LockFactory $lockFactory,
        private InvoiceService $invoiceService,
    ) {}

    public function generate(string $orderId): void
    {
        $lock = $this->lockFactory->createLock(
            resource: 'invoice_generation_' . $orderId,
            ttl: 30.0,        // auto-release after 30s (safety net)
            autoRelease: true, // release on object destruction
        );

        // Blocking: wait up to 10s for the lock
        if (!$lock->acquire(blocking: true)) {
            throw new LockTimeoutException("Could not acquire lock for order $orderId");
        }

        try {
            if ($this->invoiceService->invoiceExists($orderId)) {
                return; // another worker already generated it
            }

            $this->invoiceService->createInvoice($orderId);
        } finally {
            $lock->release();
        }
    }
}
```

### Non-Blocking Lock

```php
public function refreshCache(string $cacheKey): void
{
    $lock = $this->lockFactory->createLock('cache_refresh_' . $cacheKey, ttl: 60.0);

    // Non-blocking: fail immediately if another process holds the lock
    if (!$lock->acquire(blocking: false)) {
        $this->logger->info("Cache refresh already in progress for $cacheKey, skipping.");
        return;
    }

    try {
        $this->cacheWarmer->rebuild($cacheKey);
    } finally {
        $lock->release();
    }
}
```

### Symfony Config (Redis Store)

```yaml
# config/packages/lock.yaml
framework:
    lock:
        invoice:
            resource: '%env(REDIS_LOCK_DSN)%'
            expiring_store_ttl: 30
```

> **Safety First:** The `ttl` is your dead-man switch. If a process crashes while holding the lock, the lock auto-expires after TTL seconds. Set it too low and long-running operations lose the lock mid-execution (another worker barges in). Set it too high and a crashed process blocks all workers for that duration. Measure your 99th-percentile execution time and set TTL to 2–3x that value.

---

## Race Conditions & Upsert (`ON CONFLICT`)

**Scenario:** Two workers process SQS messages for the same vehicle simultaneously. Both check `SELECT * FROM vehicle WHERE external_id = 'v-123'` — both get zero rows — both `INSERT`. One succeeds, the other throws a unique constraint violation. Your error logs fill up, messages retry, and the problem compounds.

**Solution:** Replace the check-then-insert with an atomic `UPSERT`. The database handles the race at the storage engine level.

### The Broken Pattern

```php
// WRONG: race condition between SELECT and INSERT
$existing = $repo->findByExternalId($externalId);

if ($existing === null) {
    $repo->insert(new Vehicle($externalId, $data)); // 💥 duplicate key
} else {
    $repo->update($existing, $data);
}
```

### The Fix: Postgres Upsert

```sql
INSERT INTO vehicle (external_id, make, model, updated_at)
VALUES (:external_id, :make, :model, NOW())
ON CONFLICT (external_id)
DO UPDATE SET
    make       = EXCLUDED.make,
    model      = EXCLUDED.model,
    updated_at = NOW()
WHERE vehicle.updated_at < EXCLUDED.updated_at;
```

### Doctrine DBAL Implementation

```php
class VehicleRepository
{
    public function __construct(private Connection $connection) {}

    public function upsert(string $externalId, string $make, string $model): void
    {
        $this->connection->executeStatement(
            'INSERT INTO vehicle (external_id, make, model, updated_at)
             VALUES (:external_id, :make, :model, NOW())
             ON CONFLICT (external_id)
             DO UPDATE SET
                 make       = EXCLUDED.make,
                 model      = EXCLUDED.model,
                 updated_at = NOW()
             WHERE vehicle.updated_at <= EXCLUDED.updated_at',
            [
                'external_id' => $externalId,
                'make'        => $make,
                'model'       => $model,
            ]
        );
    }
}
```

> **Safety First:** The `WHERE vehicle.updated_at < EXCLUDED.updated_at` clause in the `DO UPDATE` prevents stale data from overwriting newer data (ties into the out-of-order message problem from earlier). Without it, the last writer always wins — even if it carries older data. Also: ensure `external_id` has a `UNIQUE` index, otherwise `ON CONFLICT` has nothing to conflict against and silently inserts duplicates.

---

## Stress Testing Concurrency

**Scenario:** You wrote the upsert fix. Your unit test passes. But does it hold under real concurrency? A single-threaded PHPUnit test won't reproduce the race condition — you need multiple processes hitting the same row simultaneously.

**Methodology:** Spawn N parallel processes, each attempting the same upsert. Collect exit codes and stdout. If any process throws an exception or the final row count is wrong, the fix is incomplete.

### Bash Test Script

```bash
#!/bin/bash
set -euo pipefail

EXTERNAL_ID="stress-test-vehicle-$(date +%s)"
CONCURRENCY=20
ERRORS=0
PIDS=()

echo "Spawning $CONCURRENCY concurrent upserts for $EXTERNAL_ID..."

for i in $(seq 1 $CONCURRENCY); do
    php bin/console app:upsert-vehicle \
        --external-id="$EXTERNAL_ID" \
        --make="Make-$i" \
        --model="Model-$i" \
        > "/tmp/upsert_${i}.log" 2>&1 &
    PIDS+=($!)
done

# Wait for all processes and collect failures
for pid in "${PIDS[@]}"; do
    if ! wait "$pid"; then
        ERRORS=$((ERRORS + 1))
    fi
done

# Verify: exactly 1 row should exist
ROW_COUNT=$(psql -t -A -c \
    "SELECT COUNT(*) FROM vehicle WHERE external_id = '$EXTERNAL_ID'")

echo "Errors: $ERRORS / $CONCURRENCY"
echo "Row count: $ROW_COUNT (expected: 1)"

if [[ "$ERRORS" -gt 0 || "$ROW_COUNT" -ne 1 ]]; then
    echo "FAIL: Concurrency test failed"
    for i in $(seq 1 $CONCURRENCY); do
        if grep -qi "exception\|error" "/tmp/upsert_${i}.log"; then
            echo "--- Process $i ---"
            cat "/tmp/upsert_${i}.log"
        fi
    done
    exit 1
fi

echo "PASS: All $CONCURRENCY processes completed, exactly 1 row."
```

### Symfony Command for the Test

```php
#[AsCommand(name: 'app:upsert-vehicle')]
class UpsertVehicleCommand extends Command
{
    public function __construct(private VehicleRepository $repo)
    {
        parent::__construct();
    }

    protected function configure(): void
    {
        $this->addOption('external-id', null, InputOption::VALUE_REQUIRED);
        $this->addOption('make', null, InputOption::VALUE_REQUIRED);
        $this->addOption('model', null, InputOption::VALUE_REQUIRED);
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $this->repo->upsert(
            $input->getOption('external-id'),
            $input->getOption('make'),
            $input->getOption('model'),
        );

        $output->writeln('OK');
        return Command::SUCCESS;
    }
}
```

> **Safety First:** Run stress tests against a dedicated test database, not staging or production. Also, 20 concurrent processes on a local machine won't reproduce network-level race conditions (where latency between app server and DB amplifies the window). For realistic testing, run this from multiple containers targeting a shared database instance.

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

---

## JS Testing with Jest

**Scenario:** Your frontend calculates a vehicle's monthly payment based on price, interest rate, and loan term. This logic lives in a pure TypeScript module — no DOM, no API calls. You need fast, isolated tests with good mocking support for the edge cases (zero interest, negative values, rounding).

### Module Under Test

```typescript
// src/finance/calculator.ts
export function monthlyPayment(
    principal: number,
    annualRate: number,
    termMonths: number
): number {
    if (termMonths <= 0) throw new Error('Term must be positive');
    if (annualRate === 0) return parseFloat((principal / termMonths).toFixed(2));

    const monthlyRate = annualRate / 12 / 100;
    const factor = Math.pow(1 + monthlyRate, termMonths);
    return parseFloat(((principal * monthlyRate * factor) / (factor - 1)).toFixed(2));
}
```

### Jest Test

```typescript
// src/finance/__tests__/calculator.test.ts
import { monthlyPayment } from '../calculator';

describe('monthlyPayment', () => {
    it('calculates standard loan payment', () => {
        // $20,000 at 5% for 60 months
        expect(monthlyPayment(20000, 5, 60)).toBe(377.42);
    });

    it('handles zero interest', () => {
        expect(monthlyPayment(12000, 0, 12)).toBe(1000.00);
    });

    it('throws on zero term', () => {
        expect(() => monthlyPayment(10000, 5, 0)).toThrow('Term must be positive');
    });
});
```

### Mocking an API Module

```typescript
// src/finance/priceService.ts
export async function fetchVehiclePrice(vehicleId: string): Promise<number> {
    const res = await fetch(`/api/v1/vehicles/${vehicleId}/price`);
    return (await res.json()).price;
}

// src/finance/__tests__/priceService.test.ts
import { fetchVehiclePrice } from '../priceService';

// Mock the global fetch
global.fetch = jest.fn();

describe('fetchVehiclePrice', () => {
    it('returns price from API', async () => {
        (fetch as jest.Mock).mockResolvedValueOnce({
            json: async () => ({ price: 25000 }),
        });

        const price = await fetchVehiclePrice('v-123');
        expect(price).toBe(25000);
        expect(fetch).toHaveBeenCalledWith('/api/v1/vehicles/v-123/price');
    });

    it('propagates network errors', async () => {
        (fetch as jest.Mock).mockRejectedValueOnce(new Error('Network failure'));

        await expect(fetchVehiclePrice('v-123')).rejects.toThrow('Network failure');
    });
});
```

> **Safety First:** Jest runs tests in parallel by default (one worker per CPU core). If your tests share mutable state (e.g., `global.fetch = jest.fn()` without cleanup), tests can leak state into each other. Always restore mocks in `afterEach` or use `jest.restoreAllMocks()`. For true isolation, use `--runInBand` to run sequentially — slower, but eliminates parallel flakiness during debugging.

---

## Out-of-Order Message Testing

**Scenario:** SQS delivers two messages for vehicle `v-123`: first `status=inspected` (timestamp `T2`), then `status=received` (timestamp `T1`, older). If your consumer blindly applies each message, the vehicle ends up as "received" when it should be "inspected." Your test must prove the consumer correctly discards stale events.

### Consumer Logic (Recap)

```php
class VehicleStatusConsumer
{
    public function __construct(private Connection $db) {}

    public function handle(array $message): void
    {
        $this->db->executeStatement(
            'INSERT INTO vehicle_status (external_id, status, event_timestamp)
             VALUES (:id, :status, :ts)
             ON CONFLICT (external_id)
             DO UPDATE SET
                 status          = EXCLUDED.status,
                 event_timestamp = EXCLUDED.event_timestamp
             WHERE vehicle_status.event_timestamp < EXCLUDED.event_timestamp',
            [
                'id'     => $message['entity_id'],
                'status' => $message['payload']['status'],
                'ts'     => $message['event_timestamp'],
            ]
        );
    }
}
```

### PHPUnit Test: Out-of-Order Delivery

```php
class VehicleStatusConsumerTest extends TestCase
{
    private Connection $db;
    private VehicleStatusConsumer $consumer;

    protected function setUp(): void
    {
        $this->db = TestDatabaseFactory::create(); // test DB with migrations applied
        $this->consumer = new VehicleStatusConsumer($this->db);
    }

    public function testNewerEventWins(): void
    {
        // Deliver T2 first (newer event arrives first)
        $this->consumer->handle([
            'entity_id'       => 'v-123',
            'event_timestamp' => '2026-03-06T14:00:00Z',
            'payload'         => ['status' => 'inspected'],
        ]);

        // Deliver T1 second (older event arrives late)
        $this->consumer->handle([
            'entity_id'       => 'v-123',
            'event_timestamp' => '2026-03-06T13:00:00Z',
            'payload'         => ['status' => 'received'],
        ]);

        // Assert: the newer status survives
        $row = $this->db->fetchAssociative(
            'SELECT status, event_timestamp FROM vehicle_status WHERE external_id = :id',
            ['id' => 'v-123']
        );

        self::assertSame('inspected', $row['status']);
        self::assertSame('2026-03-06T14:00:00Z', $row['event_timestamp']);
    }

    public function testOlderEventIsAppliedWhenNoRowExists(): void
    {
        // First message for this entity — should always insert regardless of age
        $this->consumer->handle([
            'entity_id'       => 'v-456',
            'event_timestamp' => '2026-03-06T10:00:00Z',
            'payload'         => ['status' => 'received'],
        ]);

        $row = $this->db->fetchAssociative(
            'SELECT status FROM vehicle_status WHERE external_id = :id',
            ['id' => 'v-456']
        );

        self::assertSame('received', $row['status']);
    }

    public function testSameTimestampDoesNotOverwrite(): void
    {
        $ts = '2026-03-06T14:00:00Z';

        $this->consumer->handle([
            'entity_id'       => 'v-789',
            'event_timestamp' => $ts,
            'payload'         => ['status' => 'inspected'],
        ]);

        // Same timestamp, different status — should NOT overwrite (< not <=)
        $this->consumer->handle([
            'entity_id'       => 'v-789',
            'event_timestamp' => $ts,
            'payload'         => ['status' => 'ready_for_sale'],
        ]);

        $row = $this->db->fetchAssociative(
            'SELECT status FROM vehicle_status WHERE external_id = :id',
            ['id' => 'v-789']
        );

        self::assertSame('inspected', $row['status']);
    }

    /**
     * Simulate realistic SQS delivery: randomize the order of N events
     * and assert the final state always reflects the newest event.
     */
    public function testRandomizedDeliveryOrder(): void
    {
        $events = [
            ['ts' => '2026-03-06T10:00:00Z', 'status' => 'received'],
            ['ts' => '2026-03-06T11:00:00Z', 'status' => 'inspected'],
            ['ts' => '2026-03-06T12:00:00Z', 'status' => 'ready_for_sale'],
            ['ts' => '2026-03-06T13:00:00Z', 'status' => 'reserved'],
            ['ts' => '2026-03-06T14:00:00Z', 'status' => 'sold'],
        ];

        // Run 10 times with different shuffles
        for ($i = 0; $i < 10; $i++) {
            $this->db->executeStatement("DELETE FROM vehicle_status WHERE external_id = 'v-shuffle'");
            $shuffled = $events;
            shuffle($shuffled);

            foreach ($shuffled as $event) {
                $this->consumer->handle([
                    'entity_id'       => 'v-shuffle',
                    'event_timestamp' => $event['ts'],
                    'payload'         => ['status' => $event['status']],
                ]);
            }

            $row = $this->db->fetchAssociative(
                'SELECT status, event_timestamp FROM vehicle_status WHERE external_id = :id',
                ['id' => 'v-shuffle']
            );

            self::assertSame('sold', $row['status'], "Failed on shuffle iteration $i");
            self::assertSame('2026-03-06T14:00:00Z', $row['event_timestamp']);
        }
    }
}
```

> **Safety First:** The randomized test (`testRandomizedDeliveryOrder`) is a poor man's property-based test. It catches ordering bugs, but `shuffle()` is non-deterministic — a failing run might not reproduce. Log the shuffle order on failure so you can replay the exact sequence. For deterministic reproduction, seed the shuffle: `mt_srand($seed); shuffle($shuffled);` and log the seed.
