---
layout: default
title: Race Conditions & Upsert (ON CONFLICT)
parent: Testing, Concurrency, Distributed Locks
nav_order: 2
---

## Race Conditions & Upsert ([`ON CONFLICT`](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT))

**Scenario:** Two workers process SQS messages for the same vehicle simultaneously. Both check `SELECT * FROM vehicle WHERE external_id = 'v-123'` - both get zero rows - both `INSERT`. One succeeds, the other throws a unique constraint violation. Your error logs fill up, messages retry, and the problem compounds.

**Solution:** Replace the check-then-insert with an atomic `UPSERT`. The database handles the race at the storage engine level.

### The Broken Pattern

```php
// WRONG: race condition between SELECT and INSERT
$existing = $repo->findByExternalId($externalId);

if ($existing === null) {
    $repo->insert(new Vehicle($externalId, $data)); // duplicate key
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

### [Doctrine DBAL](https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/index.html) Implementation

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

> **Safety First:** The `WHERE vehicle.updated_at < EXCLUDED.updated_at` clause in the `DO UPDATE` prevents stale data from overwriting newer data (ties into the out-of-order message problem from earlier). Without it, the last writer always wins - even if it carries older data. Also: ensure `external_id` has a `UNIQUE` index, otherwise `ON CONFLICT` has nothing to conflict against and silently inserts duplicates.
