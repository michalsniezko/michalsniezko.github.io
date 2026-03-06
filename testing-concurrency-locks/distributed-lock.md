---
layout: default
title: Distributed Lock (Symfony Lock)
parent: Testing, Concurrency, Distributed Locks
nav_order: 1
---

## Distributed Lock ([Symfony Lock](https://symfony.com/doc/current/lock.html))

**Scenario:** Two Kubernetes pods receive the same SQS message (at-least-once delivery). Both try to generate an invoice for order `ord-999`. Without coordination, you get duplicate invoices and a very confused accounting team.

A distributed lock ensures only one process executes a critical section at a time, even across multiple nodes. The lock state lives in a shared store (Redis, PostgreSQL, DynamoDB) - not in local memory.

### Blocking vs. Non-Blocking

| Strategy         | Behavior                                    | Use When                                |
|------------------|---------------------------------------------|-----------------------------------------|
| **Blocking**     | Waits (up to TTL) for the lock to release   | Work must eventually complete - e.g., invoice generation |
| **Non-Blocking** | Returns `false` immediately if lock is held | Duplicate work is safe to skip - e.g., idempotent cache refresh |

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
