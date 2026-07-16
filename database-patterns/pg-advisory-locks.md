---
layout: default
title: PostgreSQL Advisory Locks
parent: Database & Persistence
nav_order: 11
---

## PostgreSQL Advisory Locks

PostgreSQL's row-level locks (`SELECT FOR UPDATE`) protect specific rows. Advisory locks are different: they protect anything you decide to represent with an integer key. PostgreSQL stores and manages them, but what the key means is entirely up to you.

This makes advisory locks useful as a general-purpose distributed mutex - coordinating background jobs, preventing concurrent migrations, implementing leader election - without creating a separate Redis instance or any additional infrastructure.

---

## The Lock Family

All advisory lock functions take either a single `bigint` key or two `int` keys (which PostgreSQL combines internally):

```sql
-- Session-level (must release explicitly or connection closes)
pg_advisory_lock(key bigint)
pg_advisory_lock(key1 int, key2 int)
pg_advisory_lock_shared(key bigint)
pg_try_advisory_lock(key bigint) -- returns bool, never blocks

-- Transaction-level (released automatically at COMMIT/ROLLBACK)
pg_advisory_xact_lock(key bigint)
pg_advisory_xact_lock(key1 int, key2 int)
pg_advisory_xact_lock_shared(key bigint)
pg_try_advisory_xact_lock(key bigint) -- returns bool, never blocks
```

All blocking variants wait indefinitely. The `try_` variants return `true` if the lock was acquired, `false` if it was already held - never block.

Shared locks allow multiple holders simultaneously. Exclusive locks allow only one holder. Shared and exclusive variants follow the same rules as regular read/write locks.

---

## Session vs Transaction Level

This is the most important distinction.

**Transaction-level** locks are released automatically when the current transaction ends (commit or rollback). They work like row locks: you acquire inside a transaction and PostgreSQL cleans up for you.

```sql
BEGIN;
SELECT pg_advisory_xact_lock(12345);
-- ... critical section ...
COMMIT; -- lock released here automatically
```

**Session-level** locks survive transaction boundaries. They stay held until explicitly released or until the database connection closes.

```sql
SELECT pg_advisory_lock(12345);
-- ... critical section, can span multiple transactions ...
SELECT pg_advisory_unlock(12345); -- must call this explicitly
```

Session-level locks can be useful when the critical section spans multiple transactions. But they require explicit release and behave dangerously with connection pooling (covered below).

**Default to transaction-level** unless you have a specific reason for session-level.

---

## Key Space Design

Advisory locks use integers. Every application sharing a database uses the same integer space. Without a namespace strategy, two unrelated parts of the system can accidentally contend on the same integer.

A common approach: define a fixed offset per concern and add a resource ID:

```php
// A fixed offset per concern type
const LOCK_JOB_SYNC    = 100_000;
const LOCK_JOB_EXPORT  = 200_000;
const LOCK_MIGRATION   = 300_000;

// Final key = offset + resource ID
function jobLockKey(int $offset, int $resourceId): int
{
    return $offset + $resourceId;
}

// Lock for sync job on resource 42
$key = jobLockKey(LOCK_JOB_SYNC, 42); // 100042
```

Alternatively, hash a string key into a bigint:

```sql
-- Hash a string into a bigint for use as a lock key
SELECT ('job_sync_42'::text::bytea)::bigint; -- deterministic
```

In PHP:

```php
function advisoryKey(string $name): int
{
    // crc32 returns a signed 32-bit int, fits in bigint
    return crc32($name);
}

$key = advisoryKey('job:sync:42');
```

CRC32 has collision risk for large key spaces. For safety, use the two-integer form with a fixed namespace ID:

```php
// (namespaceId, resourceId) - PostgreSQL combines them internally
// pg_advisory_lock($namespaceId, $resourceId)
const NS_JOB_SYNC = 1001;
$pdo->execute('SELECT pg_advisory_xact_lock(?, ?)', [NS_JOB_SYNC, $resourceId]);
```

---

## Use Case: Preventing Concurrent Background Jobs

The most common use case. A background job should not run twice simultaneously for the same resource. Without coordination, two workers poll the queue, both pick up the same job, and both execute it.

```php
function processJob(PDO $pdo, int $jobId): void
{
    $pdo->beginTransaction();

    // Try to acquire - returns false immediately if another worker holds it
    $stmt = $pdo->prepare('SELECT pg_try_advisory_xact_lock(?)');
    $stmt->execute([jobLockKey(LOCK_JOB_SYNC, $jobId)]);
    $acquired = (bool) $stmt->fetchColumn();

    if (!$acquired) {
        $pdo->rollBack();
        return; // another worker is handling this job, skip it
    }

    // We hold the lock for the duration of this transaction
    doWork($jobId);

    $pdo->commit(); // lock released here
}
```

The lock is released at `COMMIT`. If the worker crashes mid-transaction, PostgreSQL rolls back and releases the lock. No manual cleanup needed.

---

## Use Case: Coordinating Migration Steps

Schema migrations that run from multiple application instances need coordination. Only one instance should run the migration; others should wait or skip.

```php
function runMigration(PDO $pdo, int $migrationVersion): void
{
    $pdo->beginTransaction();

    // Block until we can proceed - all other instances wait here
    $pdo->prepare('SELECT pg_advisory_xact_lock(?)')
        ->execute([jobLockKey(LOCK_MIGRATION, $migrationVersion)]);

    // Check if another instance already ran this migration
    $stmt = $pdo->prepare('SELECT 1 FROM schema_migrations WHERE version = ?');
    $stmt->execute([$migrationVersion]);

    if ($stmt->fetchColumn()) {
        $pdo->rollBack(); // already done, release lock and exit
        return;
    }

    applyMigration($pdo, $migrationVersion);

    $pdo->prepare('INSERT INTO schema_migrations (version) VALUES (?)')
        ->execute([$migrationVersion]);

    $pdo->commit();
}
```

This is a check-then-act pattern that is atomic at the database level because the advisory lock prevents any other connection from passing the lock acquisition while we are in the critical section.

---

## Use Case: Rate-Limiting Concurrent Operations

Limit how many processes can perform an operation simultaneously using shared locks. Exclusive lock = only one at a time. Shared lock = many at a time, but exclusive claim excludes all.

A simpler pattern: use a counting semaphore via a table row lock combined with an advisory lock for the check phase.

Or use advisory locks directly for leader election - only one node should run a singleton task:

```php
function tryBecomeLeader(PDO $pdo): bool
{
    // Session-level: held until this process disconnects or explicitly unlocks
    $stmt = $pdo->prepare('SELECT pg_try_advisory_lock(?)');
    $stmt->execute([LOCK_LEADER]);
    return (bool) $stmt->fetchColumn();
}

// In worker bootstrap:
if (!tryBecomeLeader($pdo)) {
    // Another worker holds the leader lock - this is a follower
    runFollowerMode();
} else {
    // This process is the leader
    runLeaderMode();
    // Lock will be released when $pdo connection closes (process exit)
}
```

When the leader process dies, its connection closes, the session lock is released, and another worker can become leader.

---

## Pitfalls

### Session locks on pooled connections

Session-level advisory locks are tied to the database connection, not the PHP request. With PgBouncer in session mode or PHP persistent connections, the same connection is reused by the next request. If you acquired a session lock and forgot to release it, the next request inherits the held lock.

```php
// Dangerous with connection pooling
pg_advisory_lock(12345);  // session-level
do_work();
// forgot pg_advisory_unlock(12345) - next request on this connection holds the lock
```

Rule: if using session-level locks, always release in a `finally` block:

```php
$pdo->prepare('SELECT pg_advisory_lock(?)')->execute([$key]);
try {
    doWork();
} finally {
    $pdo->prepare('SELECT pg_advisory_unlock(?)')->execute([$key]);
}
```

Or better: use transaction-level locks (`pg_advisory_xact_lock`) which are always cleaned up at transaction end, even on crash.

### Deadlocks are still possible

Advisory locks participate in PostgreSQL's deadlock detection. If process A holds advisory lock 1 and waits for advisory lock 2, while process B holds advisory lock 2 and waits for advisory lock 1, PostgreSQL will detect the cycle and abort one of them with:

```
ERROR: deadlock detected
DETAIL: Process X waits for ShareLock on advisory 2; blocked by process Y.
```

Avoid deadlocks by always acquiring multiple locks in the same order across all code paths.

### Blocking variants wait forever

`pg_advisory_lock()` (without `try_`) blocks indefinitely. In a web context, a hung upstream or a lock holder that never finishes can hold all available workers. Always use `pg_try_advisory_lock()` in request-handling code and handle the false return:

```php
$acquired = $pdo->prepare('SELECT pg_try_advisory_xact_lock(?)');
$acquired->execute([$key]);

if (!(bool) $acquired->fetchColumn()) {
    throw new ConcurrentOperationException('Operation already in progress');
}
```

### Key collisions between applications

If multiple applications share a PostgreSQL instance, advisory lock keys collide globally. A key `12345` in application A and key `12345` in application B contend for the same lock. Document your key ranges per application or use the two-integer form with a per-application namespace.

---

## Monitoring Held Locks

```sql
-- See all currently held advisory locks
SELECT pid, locktype, objid, classid, objsubid, granted, mode
FROM pg_locks
WHERE locktype = 'advisory';

-- Join with pg_stat_activity to see which queries hold them
SELECT
    l.pid,
    l.objid,
    l.classid,
    l.mode,
    l.granted,
    a.query,
    a.state,
    a.query_start
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.locktype = 'advisory';
```

`classid` corresponds to the first integer argument; `objid` to the second (or to the full bigint when using the single-argument form). A `granted = false` row means a session is blocked waiting for the lock.

---

### For AI agents

```
PostgreSQL advisory locks: user-defined locks keyed by integer that PostgreSQL manages. Not tied to rows - used for distributed mutexes, job deduplication, leader election. Two levels: transaction-level (pg_advisory_xact_lock - released automatically at COMMIT/ROLLBACK, prefer this) and session-level (pg_advisory_lock - must release explicitly or held until connection closes). Non-blocking variant: pg_try_advisory_xact_lock returns bool, never blocks. Key design: use a fixed namespace offset per concern + resource ID to avoid key collisions (same key space is shared across all connections). Critical pitfall: session-level locks on pooled connections leak to the next request that reuses the connection - always use finally block to release, or switch to transaction-level. Deadlock detection works on advisory locks. Monitor via pg_locks WHERE locktype = 'advisory'; blocked waiters show granted = false.
```

Reference: `https://michalsniezko.github.io/database-patterns/pg-advisory-locks.html`
