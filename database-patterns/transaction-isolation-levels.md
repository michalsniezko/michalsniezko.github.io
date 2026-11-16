---
layout: default
title: Transaction Isolation Levels
parent: Database & Persistence
nav_order: 15
---

## Transaction Isolation Levels

When two transactions run at the same time, how much can one see of the other? The answer is the **isolation level**. A higher level hides more of the other transactions' work, so your transaction behaves more as if it ran alone. The cost is more failed transactions that you must retry.

Most applications never set the isolation level. They run on the database default and do not know which problems that default allows. This article explains the problems, the four levels, and what PostgreSQL really does at each one.

Isolation levels work together with locks. Locks are covered in [PostgreSQL Lock Types](../database-patterns/postgres-lock-types.md).

---

## The Problems

Each isolation level is defined by which of these problems it allows. All examples use two transactions, A and B, running at the same time.

### Dirty read

A reads data that B wrote but has **not committed yet**. If B rolls back, A has used data that never existed.

```
B: UPDATE account SET balance = 0 WHERE id = 1;     -- not committed
A: SELECT balance FROM account WHERE id = 1;         -- sees 0
B: ROLLBACK;                                          -- balance is 100 again
```

### Non-repeatable read

A reads the same row twice and gets two different values, because B committed a change in between.

```
A: SELECT balance FROM account WHERE id = 1;         -- 100
B: UPDATE account SET balance = 50 WHERE id = 1; COMMIT;
A: SELECT balance FROM account WHERE id = 1;         -- 50
```

### Phantom read

A runs the same query twice and gets a different **set of rows**, because B inserted or deleted rows that match the condition.

```
A: SELECT count(*) FROM orders WHERE customer_id = 7;   -- 3
B: INSERT INTO orders (customer_id, ...) VALUES (7, ...); COMMIT;
A: SELECT count(*) FROM orders WHERE customer_id = 7;   -- 4
```

### Lost update

A and B both read a value, both compute a new value from it, both write. The second write overwrites the first, and one change is lost.

```
A: SELECT stock FROM product WHERE id = 1;    -- 10
B: SELECT stock FROM product WHERE id = 1;    -- 10
A: UPDATE product SET stock = 9 WHERE id = 1; COMMIT;
B: UPDATE product SET stock = 9 WHERE id = 1; COMMIT;   -- should be 8
```

### Write skew

A and B read the same data, make a decision based on it, and each write a **different** row. Each write is fine alone. Together they break a rule.

The classic example: at least one doctor must be on call. Two doctors are on call, and both ask to go off call at the same time.

```
A: SELECT count(*) FROM doctor WHERE on_call;    -- 2, so I can leave
B: SELECT count(*) FROM doctor WHERE on_call;    -- 2, so I can leave
A: UPDATE doctor SET on_call = false WHERE name = 'Alice'; COMMIT;
B: UPDATE doctor SET on_call = false WHERE name = 'Bob';   COMMIT;
-- Nobody is on call
```

No row was written twice, so row locks do not help. This is the hardest problem to notice.

---

## The Four Levels

The SQL standard defines four levels:

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| `READ UNCOMMITTED` | possible | possible | possible |
| `READ COMMITTED` | prevented | possible | possible |
| `REPEATABLE READ` | prevented | prevented | possible |
| `SERIALIZABLE` | prevented | prevented | prevented |

The standard says what a level must **prevent**. A database may prevent more. PostgreSQL does:

| Level in PostgreSQL | Dirty read | Non-repeatable read | Phantom read | Lost update | Write skew |
|---|---|---|---|---|---|
| `READ UNCOMMITTED` | prevented | possible | possible | possible | possible |
| `READ COMMITTED` (default) | prevented | possible | possible | possible | possible |
| `REPEATABLE READ` | prevented | prevented | prevented | prevented (error) | possible |
| `SERIALIZABLE` | prevented | prevented | prevented | prevented (error) | prevented (error) |

"(error)" means PostgreSQL does not silently allow the problem. It cancels one of the transactions with an error, and you must retry it.

---

## READ UNCOMMITTED

PostgreSQL accepts this level but treats it exactly like `READ COMMITTED`. You can never see uncommitted data in PostgreSQL, at any level.

---

## READ COMMITTED (the PostgreSQL default)

Each **statement** sees a snapshot of data that was committed when that statement started. Two statements in the same transaction can see different data.

This is why non-repeatable reads, phantoms, and lost updates are possible: between your `SELECT` and your `UPDATE`, other transactions commit.

### UPDATE re-checks the row

There is one detail that makes `READ COMMITTED` safer than it looks. When an `UPDATE` finds a row that another transaction is changing, it waits. When the other transaction commits, PostgreSQL takes the **new** version of the row, checks the `WHERE` condition again, and applies the update to the new version.

So this is safe in `READ COMMITTED`, even with many parallel requests:

```sql
UPDATE product SET stock = stock - 1 WHERE id = 1 AND stock > 0;
```

But this is not, because the computing happens in the application, on an old value:

```php
$stock = $connection->fetchOne('SELECT stock FROM product WHERE id = ?', [1]);
$connection->executeStatement('UPDATE product SET stock = ? WHERE id = ?', [$stock - 1, 1]);
```

In `READ COMMITTED`, fix the second form with `SELECT ... FOR UPDATE` or with optimistic locking. Both are covered in [PostgreSQL Lock Types](../database-patterns/postgres-lock-types.md).

---

## REPEATABLE READ

The whole **transaction** sees one snapshot, taken at its first statement (not at `BEGIN`). Every query in the transaction sees the database as it was at that moment. Changes committed later by others are invisible.

This prevents non-repeatable reads, and in PostgreSQL also phantoms. It is a good fit for reports and exports that run many queries and need them to agree with each other.

### Write conflicts become errors

If your transaction tries to update a row that another transaction changed and committed **after your snapshot**, PostgreSQL does not re-check the row as in `READ COMMITTED`. Your snapshot is out of date, so it cancels your transaction:

```
ERROR:  could not serialize access due to concurrent update
```

The SQLSTATE error code is `40001`. This is how `REPEATABLE READ` prevents lost updates: the second writer fails instead of overwriting. The application must retry the whole transaction from the start.

### Write skew is still possible

In the doctors example, A and B update **different** rows. There is no write conflict, so `REPEATABLE READ` lets both commit. Only `SERIALIZABLE` catches this.

---

## SERIALIZABLE

The result is guaranteed to be the same as if the transactions had run one after another, in some order. Every problem in the list above is prevented, including write skew.

PostgreSQL does this with SSI (Serializable Snapshot Isolation). It works like `REPEATABLE READ`, and also tracks what each transaction has read. When it finds that two transactions read and wrote in a way that no serial order could produce, it cancels one:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
```

Same SQLSTATE: `40001`.

Important points:

- `SERIALIZABLE` in PostgreSQL does **not** block more. It **fails** more. Readers still do not wait for writers.
- Every transaction at this level can fail with `40001`, even a read-only one. Retry is mandatory, not optional.
- To make this work well, every transaction that touches the same data must use `SERIALIZABLE`. A `READ COMMITTED` transaction is not checked.
- The tracking shows up in `pg_locks` as `SIReadLock`. These are not real locks and never block anything.
- Good indexes reduce false failures. Without an index, PostgreSQL tracks reads on the whole table, and unrelated transactions start to conflict.

For long read-only reports, use:

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
```

`DEFERRABLE` may wait a moment at the start, but after that the transaction can never fail with a serialization error.

---

## Setting the Level

For one transaction:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
-- ...
COMMIT;

-- or, as the first statement inside the transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

For all transactions of a session, or as the server default:

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
ALTER DATABASE app SET default_transaction_isolation = 'repeatable read';
```

In Doctrine DBAL:

```php
use Doctrine\DBAL\TransactionIsolationLevel;

$connection->setTransactionIsolation(TransactionIsolationLevel::SERIALIZABLE);
```

On PostgreSQL this runs `SET SESSION CHARACTERISTICS ...`, so the level applies to every following transaction on that connection, not only the next one.

### Careful with PgBouncer

With PgBouncer in transaction pooling mode, each transaction can run on a different server connection. A `SET SESSION ...` lands on one server connection, and your next transaction may run on another one with the default level. Worse, a later request from another part of the application may get the connection you changed.

With transaction pooling, set the level **inside** each transaction (`BEGIN ISOLATION LEVEL ...`), never per session. See [PHP Persistent Database Connections](../database-patterns/php-persistent-connections.md) for more on PgBouncer modes.

---

## Retrying Failed Transactions

At `REPEATABLE READ` and `SERIALIZABLE`, failed transactions are normal. Deadlocks (`40P01`) can happen at any level. Both errors mean "run the whole transaction again".

Retry the **whole** transaction, including the reads. Retrying only the failed statement reuses data from the old snapshot, which is the reason it failed.

```php
use Doctrine\DBAL\Connection;
use Doctrine\DBAL\Exception\RetryableException;

function runInTransactionWithRetry(Connection $connection, callable $work, int $maxAttempts = 3): mixed
{
    for ($attempt = 1; ; $attempt++) {
        try {
            return $connection->transactional($work);
        } catch (RetryableException $exception) {
            if ($attempt >= $maxAttempts) {
                throw $exception;
            }
            usleep(random_int(10_000, 50_000) * $attempt); // wait a bit, with randomness
        }
    }
}
```

Doctrine DBAL maps PostgreSQL deadlocks and serialization failures to exceptions that implement `RetryableException`.

This helper is for DBAL code. If the work uses the Doctrine ORM, an exception inside a transaction closes the EntityManager, so the second attempt fails. With the ORM, reset the EntityManager (for example with `ManagerRegistry::resetManager()`) and load the entities again before each retry.

Keep side effects out of the retried function. Sending an email or publishing a message inside it means it may happen two or three times.

---

## Other Databases Differ

The level names are the same everywhere. The behavior is not.

- **MySQL InnoDB** uses `REPEATABLE READ` as the default, not `READ COMMITTED`. Plain `SELECT` reads a snapshot, but `UPDATE`, `DELETE`, and locking reads work on the newest data and take gap locks to prevent phantoms. These extra locks cause more waiting and more deadlocks than PostgreSQL at the same level.
- **SQL Server** and **Oracle** use `READ COMMITTED` as the default, but Oracle has no `REPEATABLE READ`, and SQL Server can use either locks or row versions depending on settings.

When you move code between databases, or read an article written for another database, check what the level really does there.

---

## Which Level to Use

| Situation | Level |
|---|---|
| Normal web requests, single-row updates done in SQL | `READ COMMITTED` |
| Read, compute in PHP, write one row | `READ COMMITTED` + `FOR UPDATE` or optimistic locking |
| Report or export with many queries that must agree | `REPEATABLE READ` (or `SERIALIZABLE READ ONLY DEFERRABLE`) |
| Business rule across several rows (write skew risk) | `SERIALIZABLE` with retry |
| Money moving between accounts | `SERIALIZABLE` with retry, or explicit row locks in a fixed order |

`READ COMMITTED` is a good default. Move up to a higher level for a specific transaction when you know which problem you are solving, and always add retry when you do.

---

### For AI agents

```
Transaction isolation levels (PostgreSQL). Problems: dirty read, non-repeatable read, phantom read, lost update, write skew (two transactions read the same data and each update a different row, breaking a rule). PostgreSQL never allows dirty reads; READ UNCOMMITTED = READ COMMITTED. READ COMMITTED (default): snapshot per statement; lost updates possible for read-in-app-then-write; UPDATE re-checks WHERE on the newest row version, so UPDATE ... SET x = x - 1 WHERE ... is safe. REPEATABLE READ: one snapshot per transaction (taken at first statement); no phantoms in PostgreSQL; updating a row changed after the snapshot fails with 40001 "could not serialize access due to concurrent update"; write skew still possible. SERIALIZABLE (SSI): prevents write skew; does not block more, fails more with 40001; every transaction touching the data must use it; any transaction can fail, including read-only; use READ ONLY DEFERRABLE for long reports; SIReadLock in pg_locks is not a blocking lock. Always retry the whole transaction (including reads) on 40001 and 40P01 (deadlock); in Doctrine DBAL catch RetryableException; keep side effects (emails, messages) outside the retried code. Set level per transaction with BEGIN ISOLATION LEVEL ...; Doctrine setTransactionIsolation sets it for the whole session - with PgBouncer transaction pooling never set it per session. MySQL InnoDB default is REPEATABLE READ with gap locks - behavior differs from PostgreSQL. Default choice: READ COMMITTED + FOR UPDATE or optimistic locking; SERIALIZABLE with retry for rules across several rows.
```

Reference: `https://michalsniezko.github.io/database-patterns/transaction-isolation-levels.html`
