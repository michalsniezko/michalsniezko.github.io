---
layout: default
title: PostgreSQL Lock Types
parent: Database & Persistence
nav_order: 14
---

## PostgreSQL Lock Types

Every query in PostgreSQL takes locks, even a plain `SELECT`. Most of the time you never notice, because most locks do not block each other. You notice when a migration freezes the whole application, when two workers process the same job, or when the logs fill with `deadlock detected`.

PostgreSQL has three main kinds of locks:

- **Table locks** - taken automatically by every statement on the whole table.
- **Row locks** - taken on single rows by `UPDATE`, `DELETE`, and `SELECT ... FOR UPDATE`.
- **Advisory locks** - locks on a number that the application chooses. Covered in [PostgreSQL Advisory Locks](../database-patterns/pg-advisory-locks.md).

On top of these, the application can use **optimistic locking**, which holds no lock at all. It is covered at the end.

Locks are one half of how PostgreSQL handles concurrent transactions. The other half is the isolation level, covered in [Transaction Isolation Levels](../database-patterns/transaction-isolation-levels.md).

---

## Readers Do Not Block Writers

First, the most important rule. PostgreSQL uses MVCC (Multi-Version Concurrency Control). An `UPDATE` does not change a row in place. It writes a new version of the row, and the old version stays until no transaction needs it.

Because of this:

- A plain `SELECT` never waits for an `UPDATE` or `DELETE`. It reads the old version.
- An `UPDATE` never waits for a plain `SELECT`.
- Writers wait only for other writers on the **same row**.

Most locking problems come from the few cases where this rule does not help: schema changes, and two writers on the same row.

---

## Table Locks

Every statement takes a lock on each table it touches. There are eight modes. You almost never take them by hand. The statement picks the mode.

| Mode | Taken by |
|---|---|
| `ACCESS SHARE` | `SELECT` |
| `ROW SHARE` | `SELECT ... FOR UPDATE` / `FOR SHARE` |
| `ROW EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| `SHARE UPDATE EXCLUSIVE` | `VACUUM` (not `FULL`), `ANALYZE`, `CREATE INDEX CONCURRENTLY`, some `ALTER TABLE` forms |
| `SHARE` | `CREATE INDEX` (without `CONCURRENTLY`) |
| `SHARE ROW EXCLUSIVE` | `CREATE TRIGGER`, some `ALTER TABLE` forms |
| `EXCLUSIVE` | `REFRESH MATERIALIZED VIEW CONCURRENTLY` |
| `ACCESS EXCLUSIVE` | `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`, `CLUSTER`, most `ALTER TABLE` forms, `LOCK TABLE` |

The names are confusing. "ROW EXCLUSIVE" is a table lock, not a row lock. Do not try to guess from the name. Look at what conflicts.

### What blocks what

An `X` means the two modes cannot be held at the same time by different transactions:

| Requested \ Held | ACCESS SHARE | ROW SHARE | ROW EXCL. | SHARE UPD. EXCL. | SHARE | SHARE ROW EXCL. | EXCL. | ACCESS EXCL. |
|---|---|---|---|---|---|---|---|---|
| ACCESS SHARE | | | | | | | | X |
| ROW SHARE | | | | | | | X | X |
| ROW EXCLUSIVE | | | | | X | X | X | X |
| SHARE UPDATE EXCLUSIVE | | | | X | X | X | X | X |
| SHARE | | | X | X | | X | X | X |
| SHARE ROW EXCLUSIVE | | | X | X | X | X | X | X |
| EXCLUSIVE | | X | X | X | X | X | X | X |
| ACCESS EXCLUSIVE | X | X | X | X | X | X | X | X |

What this means in practice:

- Normal traffic (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) never conflicts with itself at the table level.
- `CREATE INDEX` takes `SHARE`, which blocks all writes, but not reads. This is why [`CREATE INDEX CONCURRENTLY`](../database-patterns/concurrent-index-migrations.md) exists.
- `ACCESS EXCLUSIVE` blocks everything, even `SELECT`.

### The lock queue trap

This is the most common way a small migration makes production stop answering requests:

```sql
ALTER TABLE orders ADD COLUMN note text;
```

Adding a nullable column is fast. It only changes the table definition. But it needs `ACCESS EXCLUSIVE`, and it waits in a queue:

1. A long report query holds `ACCESS SHARE` on `orders` for 5 minutes.
2. The `ALTER TABLE` asks for `ACCESS EXCLUSIVE`. It conflicts with the report, so it waits.
3. Every new `SELECT` on `orders` asks for `ACCESS SHARE`. That conflicts with the **waiting** `ACCESS EXCLUSIVE`, so the new `SELECT` waits behind it.
4. For 5 minutes, nobody can read `orders`. The application times out.

A new lock request that conflicts with a request already waiting in the queue must wait behind it, even if it does not conflict with the lock that is actually held.

The fix is a short `lock_timeout`. If the lock is not free quickly, fail and try again later, instead of blocking everyone:

```sql
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN note text;
```

If it fails with `canceling statement due to lock timeout`, run it again. Most migration tools can retry this for you.

---

## Row Locks

Row locks are taken on single rows. There are four modes:

| Mode | Taken by |
|---|---|
| `FOR UPDATE` | `DELETE`, `UPDATE` that changes a key column, `SELECT ... FOR UPDATE` |
| `FOR NO KEY UPDATE` | `UPDATE` that does not change a key column, `SELECT ... FOR NO KEY UPDATE` |
| `FOR SHARE` | `SELECT ... FOR SHARE` |
| `FOR KEY SHARE` | Foreign key checks (`INSERT` into a child table locks the parent row), `SELECT ... FOR KEY SHARE` |

A "key column" here means a column with a unique index that a foreign key could point to, such as the primary key.

### What blocks what

| Requested \ Held | FOR KEY SHARE | FOR SHARE | FOR NO KEY UPDATE | FOR UPDATE |
|---|---|---|---|---|
| FOR KEY SHARE | | | | X |
| FOR SHARE | | | X | X |
| FOR NO KEY UPDATE | | X | X | X |
| FOR UPDATE | X | X | X | X |

The split between `FOR UPDATE` and `FOR NO KEY UPDATE` exists for foreign keys. Inserting an order line takes `FOR KEY SHARE` on the parent order. A normal `UPDATE orders SET status = ...` takes `FOR NO KEY UPDATE`. They do not conflict, so you can add lines to an order while another transaction updates its status.

Remember: none of these block a plain `SELECT`.

### SELECT ... FOR UPDATE: read, then write safely

The classic race condition: read a value, compute in the application, write it back.

```php
// Two requests run this at the same time. Both read 100, both write 90.
$balance = $connection->fetchOne('SELECT balance FROM account WHERE id = ?', [$id]);
$connection->executeStatement('UPDATE account SET balance = ? WHERE id = ?', [$balance - 10, $id]);
```

`FOR UPDATE` locks the row when you read it. The second request waits at the `SELECT` until the first commits, and then reads the new value:

```php
$connection->beginTransaction();
try {
    $balance = $connection->fetchOne(
        'SELECT balance FROM account WHERE id = ? FOR UPDATE',
        [$id],
    );
    $connection->executeStatement(
        'UPDATE account SET balance = ? WHERE id = ?',
        [$balance - 10, $id],
    );
    $connection->commit();
} catch (\Throwable $exception) {
    $connection->rollBack();
    throw $exception;
}
```

A row lock is released only at `COMMIT` or `ROLLBACK`. Outside a transaction, each statement is its own transaction, so `FOR UPDATE` without `BEGIN` releases the lock right after the `SELECT` and protects nothing.

If the change can be done in SQL, it is simpler to skip the read:

```sql
UPDATE account SET balance = balance - 10 WHERE id = 42;
```

### NOWAIT and SKIP LOCKED

By default, a row lock request waits until the row is free. Two options change that:

```sql
-- Fail at once if the row is locked
SELECT * FROM account WHERE id = 42 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "account"

-- Ignore locked rows and return only the free ones
SELECT * FROM task
WHERE status = 'pending'
ORDER BY id
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

`NOWAIT` fits user requests: tell the user "someone else is editing this" instead of hanging.

`SKIP LOCKED` is the common way to build a job queue on a PostgreSQL table. Many workers run the same query, and each one gets a different set of rows, with no waiting and no duplicates. For more on queues stored in a database table, see [Database-Backed Task Queue](../backend-patterns-optimization/database-backed-task-queue.md).

### Where row locks are stored

Row locks are written into the row itself on disk, not into shared memory. So locking a million rows does not run out of lock memory. It does mean that row locks are **not** listed in `pg_locks`, unless a transaction is waiting for one. To see which rows are locked right now, use the `pgrowlocks` extension.

---

## Deadlocks

A deadlock happens when two transactions each wait for a lock the other one holds:

```
Transaction A: UPDATE account SET ... WHERE id = 1;   -- locks row 1
Transaction B: UPDATE account SET ... WHERE id = 2;   -- locks row 2
Transaction A: UPDATE account SET ... WHERE id = 2;   -- waits for B
Transaction B: UPDATE account SET ... WHERE id = 1;   -- waits for A
```

Neither can continue. After `deadlock_timeout` (default 1 second), PostgreSQL checks for a cycle, picks one transaction, and cancels it:

```
ERROR:  deadlock detected
DETAIL: Process 4211 waits for ShareLock on transaction 9012; blocked by process 4215.
        Process 4215 waits for ShareLock on transaction 9011; blocked by process 4211.
```

The SQLSTATE error code is `40P01`.

How to avoid them:

- **Lock rows in the same order everywhere.** If every code path locks accounts by ascending ID, a cycle cannot happen:

  ```sql
  SELECT * FROM account WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
  ```

- **Keep transactions short.** No HTTP calls or slow work while holding locks.
- **Retry.** A deadlock is not a bug in the data. The cancelled transaction can simply run again. See the retry example in [Transaction Isolation Levels](../database-patterns/transaction-isolation-levels.md).

---

## Pessimistic vs Optimistic Locking

Everything above is **pessimistic locking**: you expect a conflict, so you lock first and then work. Other transactions wait.

**Optimistic locking** expects conflicts to be rare. It takes no lock. Each row has a `version` column. The update only succeeds if the version is still the one you read:

```sql
-- Read
SELECT id, status, version FROM orders WHERE id = 42;   -- version = 7

-- Write, only if nobody changed the row since
UPDATE orders
SET status = 'shipped', version = version + 1
WHERE id = 42 AND version = 7;
```

If the update changes 0 rows, someone else wrote first. The application shows an error, or reloads and tries again.

In Doctrine ORM, add a version field:

```php
#[ORM\Entity]
class Order
{
    #[ORM\Version]
    #[ORM\Column(type: 'integer')]
    private int $version;
}
```

Doctrine adds `AND version = ?` to every update, and throws `OptimisticLockException` when 0 rows change.

Doctrine also supports pessimistic locks:

```php
use Doctrine\DBAL\LockMode;

$entityManager->wrapInTransaction(function () use ($entityManager, $orderId) {
    $order = $entityManager->find(Order::class, $orderId, LockMode::PESSIMISTIC_WRITE); // FOR UPDATE
    // ...
});
```

`PESSIMISTIC_WRITE` must run inside a transaction. Without one, Doctrine throws `TransactionRequiredException`.

**When to use which:**

- **Optimistic** - a user edits a form for a few minutes, then saves. Holding a database lock that long is not possible. Conflicts are rare and the user can retry.
- **Pessimistic** - short, high-contention operations inside one request or job: balances, stock counts, job claiming. Retrying would happen too often.

---

## Seeing Who Blocks Whom

```sql
-- Every waiting query, and the query that blocks it
SELECT
    waiting.pid            AS waiting_pid,
    waiting.query          AS waiting_query,
    blocking.pid           AS blocking_pid,
    blocking.query         AS blocking_query,
    blocking.state         AS blocking_state,
    now() - blocking.xact_start AS blocking_transaction_age
FROM pg_stat_activity waiting
JOIN LATERAL unnest(pg_blocking_pids(waiting.pid)) AS blocker(pid) ON true
JOIN pg_stat_activity blocking ON blocking.pid = blocker.pid;
```

If `blocking_state` is `idle in transaction`, the blocker is an application that opened a transaction and never finished it. That is a bug in the application, and the usual cause of long lock waits. Set `idle_in_transaction_session_timeout` so PostgreSQL closes such sessions on its own.

To stop the blocker by hand:

```sql
SELECT pg_cancel_backend(4215);     -- cancel the current query
SELECT pg_terminate_backend(4215);  -- close the whole connection
```

To log every lock wait that takes longer than `deadlock_timeout`:

```sql
ALTER SYSTEM SET log_lock_waits = on;
SELECT pg_reload_conf();
```

---

### For AI agents

```
PostgreSQL locks. MVCC: plain SELECT never blocks and is never blocked by UPDATE/DELETE; writers only wait for writers on the same row. Table locks (8 modes, taken automatically): SELECT = ACCESS SHARE, INSERT/UPDATE/DELETE = ROW EXCLUSIVE, CREATE INDEX = SHARE (blocks writes), CREATE INDEX CONCURRENTLY/VACUUM/ANALYZE = SHARE UPDATE EXCLUSIVE, most ALTER TABLE/DROP/TRUNCATE/VACUUM FULL = ACCESS EXCLUSIVE (blocks even SELECT). Lock queue trap: a waiting ACCESS EXCLUSIVE blocks all later SELECTs; always SET lock_timeout (e.g. 3s) before DDL in migrations and retry. Row locks: FOR UPDATE (DELETE, key-changing UPDATE), FOR NO KEY UPDATE (normal UPDATE), FOR SHARE, FOR KEY SHARE (FK checks); FOR NO KEY UPDATE and FOR KEY SHARE do not conflict. Row locks last until COMMIT/ROLLBACK, so SELECT FOR UPDATE needs an explicit transaction. Prefer UPDATE ... SET x = x - 10 over read-modify-write in app code. NOWAIT fails at once; SKIP LOCKED for job queues. Row locks are not in pg_locks unless someone waits (use pgrowlocks). Deadlocks: SQLSTATE 40P01, detected after deadlock_timeout (1s); lock rows in a fixed order (ORDER BY id FOR UPDATE), keep transactions short, retry. Optimistic locking: version column, UPDATE ... WHERE version = ?, 0 rows = conflict (Doctrine #[ORM\Version], OptimisticLockException) - use for long user edits; pessimistic (FOR UPDATE, Doctrine LockMode::PESSIMISTIC_WRITE inside a transaction) for short high-contention work. Find blockers with pg_blocking_pids(); idle in transaction blockers = app bug, set idle_in_transaction_session_timeout; enable log_lock_waits.
```

Reference: `https://michalsniezko.github.io/database-patterns/postgres-lock-types.html`
