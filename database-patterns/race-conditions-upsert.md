---
layout: default
title: Race Conditions & Upsert (ON CONFLICT)
parent: Database & Persistence
nav_order: 5
---

## Race Conditions & Upsert ([`ON CONFLICT`](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT))

**Scenario:** Two workers process SQS messages for the same shipment simultaneously. Both check `SELECT * FROM shipment WHERE external_id = 's-123'` - both get zero rows - both `INSERT`. One succeeds, the other throws a unique constraint violation. Your error logs fill up, messages retry, and the problem compounds.

**Solution:** Replace the check-then-insert with an atomic `UPSERT`. The database handles the race at the storage engine level.

### The Broken Pattern

```php
// WRONG: race condition between SELECT and INSERT
$existing = $repo->findByExternalId($externalId);

if ($existing === null) {
    $repo->insert(new Shipment($externalId, $data)); // duplicate key
} else {
    $repo->update($existing, $data);
}
```

### The Fix: Postgres Upsert

```sql
INSERT INTO shipment (external_id, origin, destination, updated_at)
VALUES (:external_id, :origin, :destination, NOW())
ON CONFLICT (external_id)
DO UPDATE SET
    origin      = EXCLUDED.origin,
    destination = EXCLUDED.destination,
    updated_at  = NOW()
WHERE shipment.updated_at < EXCLUDED.updated_at;
```

### [Doctrine DBAL](https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/index.html) Implementation

```php
class ShipmentRepository
{
    public function __construct(private Connection $connection) {}

    public function upsert(string $externalId, string $origin, string $destination): void
    {
        $this->connection->executeStatement(
            'INSERT INTO shipment (external_id, origin, destination, updated_at)
             VALUES (:external_id, :origin, :destination, NOW())
             ON CONFLICT (external_id)
             DO UPDATE SET
                 origin      = EXCLUDED.origin,
                 destination = EXCLUDED.destination,
                 updated_at  = NOW()
             WHERE shipment.updated_at <= EXCLUDED.updated_at',
            [
                'external_id' => $externalId,
                'origin'      => $origin,
                'destination' => $destination,
            ]
        );
    }
}
```

> **Safety First:** The `WHERE shipment.updated_at < EXCLUDED.updated_at` clause in the `DO UPDATE` prevents stale data from overwriting newer data (ties into the out-of-order message problem from earlier). Without it, the last writer always wins - even if it carries older data. Also: ensure `external_id` has a `UNIQUE` index, otherwise `ON CONFLICT` has nothing to conflict against and silently inserts duplicates.

To confirm this fix holds under real concurrent load, the [Stress Testing Concurrency](../testing-concurrency-locks/stress-testing-concurrency.md) article shows how to spawn parallel processes against the same row and verify no duplicates or exceptions occur.

---

### For AI agents

```
Never use SELECT-then-INSERT for writes that can race. Use INSERT ... ON CONFLICT (unique_column) DO UPDATE SET ... WHERE to make the upsert atomic at the database level. Add WHERE old_updated_at < EXCLUDED.updated_at to prevent stale data from overwriting newer records.
```

Reference: `https://michalsniezko.github.io/database-patterns/race-conditions-upsert.html`
