---
layout: default
title: Full Load vs Incremental Load
parent: Database & Persistence
nav_order: 13
---

## Full Load vs Incremental Load

Most backend teams copy data from the production database (for example PostgreSQL) into a data warehouse (for example Amazon Redshift). Reports, dashboards, and BI (Business Intelligence) tools read the warehouse, so heavy analytics queries do not hit production.

There are two basic ways to do this copy:

- **Full load** - every run copies the whole table again and replaces the old copy.
- **Incremental load** - every run copies only the rows that are new or changed since the last run.

Both are simple to describe. The difference shows up later: one is slow but always correct, the other is fast but can quietly miss changes.

---

## Full Load

Every run, the job reads the whole source table and rebuilds the warehouse copy from scratch.

```sql
-- Source (PostgreSQL)
SELECT * FROM payment_history;
```

**Good:**
- Simple. There is no state to remember between runs.
- Always correct after a run. Updates, deletes, and manual SQL fixes are all picked up, because the copy is rebuilt.

**Bad:**
- Slow and heavy. A table with 10 million rows is read in full every time, even if only 2,000 rows changed.
- Load on the production database grows with the table size, not with the number of changes.
- The run time grows every month as the table grows.

### Do not leave an empty table during the load

The obvious way is "delete everything, then insert everything". While the insert runs, anyone who reads the table sees no rows or half the rows.

In Redshift there is one more trap: `TRUNCATE` commits the current transaction right away. You cannot wrap `TRUNCATE` + `INSERT` in one transaction and roll back if the insert fails.

Load into a staging table and swap it in at the end:

```sql
-- Redshift
CREATE TABLE payment_history_new (LIKE payment_history);

COPY payment_history_new
FROM 's3://my-bucket/exports/payment_history/'
IAM_ROLE 'arn:aws:iam::123456789012:role/redshift-copy'
FORMAT AS PARQUET;

BEGIN;
ALTER TABLE payment_history RENAME TO payment_history_old;
ALTER TABLE payment_history_new RENAME TO payment_history;
COMMIT;

DROP TABLE payment_history_old;
```

Readers see either the full old copy or the full new copy, never a half-loaded one. If the load fails, the old table is still there.

One catch: a normal Redshift view is bound to the table itself, not to its name. After the rename, such a view still reads `payment_history_old`, and the `DROP TABLE` fails because the view depends on it. Create views on swapped tables as late-binding views (`CREATE VIEW ... WITH NO SCHEMA BINDING`), which look up the table by name when they run.

---

## Incremental Load

Every run, the job copies only rows that changed since the last run. It uses a column that tells when a row was last changed, usually `updated_at`:

```sql
-- Source (PostgreSQL)
SELECT *
FROM payment
WHERE updated_at > :last_watermark;
```

The value the job remembers between runs (`last_watermark`) is called the **watermark**.

The job then adds the new rows to the warehouse copy, and updates the rows that are already there (an **upsert**).

**Good:**
- Fast and light. A run might copy a few thousand rows instead of millions.
- Load on production depends on how many rows changed, not on table size.

**Bad:**
- It only works if `updated_at` can be trusted. It misses changes in two cases:
  - A row is changed but `updated_at` does not move. For example, a manual SQL fix, or a code path that writes with raw SQL and forgets the column.
  - A row is deleted. A deleted row is gone, so `WHERE updated_at > ...` can never return it. The warehouse keeps the old row forever.

These misses are silent. The job reports success, and the warehouse is wrong.

---

## Making Incremental Load Safe

### 1. Let the database set updated_at, not the application

If `updated_at` is set in application code, every code path must remember it. A trigger makes it automatic, including manual SQL fixes:

```sql
-- PostgreSQL
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER payment_set_updated_at
BEFORE UPDATE ON payment
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

A trigger still does not help with deletes (see below).

### 2. Index the column

Without an index, `WHERE updated_at > ...` reads the whole table anyway, and you lose the main benefit of incremental load.

```sql
CREATE INDEX CONCURRENTLY idx_payment_updated_at ON payment (updated_at);
```

See [Index Migrations - Normal vs CONCURRENTLY](../database-patterns/concurrent-index-migrations.md) for why `CONCURRENTLY` matters on a live table.

### 3. Take the watermark from the data, not from the clock

Do not save "the time the job ran" as the watermark. The job server clock and the database clock can differ. Save the highest `updated_at` you actually loaded:

```sql
SELECT max(updated_at) FROM payment_staging; -- store this as the next watermark
```

### 4. Go back a few minutes on every run

In PostgreSQL, `now()` returns the time the **transaction started**, not the time it committed. Picture this:

1. 10:00:00 - a transaction starts and updates a row. `updated_at` = 10:00:00.
2. 10:00:05 - the load job runs. The transaction is not committed yet, so the job does not see the row. The job saves watermark 10:00:04 from another row.
3. 10:00:10 - the transaction commits.
4. Next run reads `updated_at > 10:00:04`. The row has 10:00:00. It is never loaded.

The fix is an overlap window: read a bit more than needed.

```sql
SELECT *
FROM payment
WHERE updated_at > :last_watermark - interval '10 minutes';
```

Pick a window longer than your longest normal transaction.

### 5. Make the load idempotent

The overlap window means the same row arrives more than once. The load must give the same result no matter how many times a row is loaded (it must be **idempotent**).

Redshift does not enforce primary keys. A plain `INSERT` will happily create duplicate rows. Use `MERGE`, or delete-then-insert from a staging table:

```sql
-- Redshift
BEGIN;

DELETE FROM payment
USING payment_staging
WHERE payment.id = payment_staging.id;

INSERT INTO payment
SELECT * FROM payment_staging;

COMMIT;
```

---

## Handling Deletes

Incremental load by `updated_at` cannot see deletes. The common options:

- **Soft delete.** Do not delete rows. Set `deleted_at` instead. That is an update, so `updated_at` moves and the change is copied. Reports filter on `deleted_at IS NULL`.
- **Periodic full reload.** Run incremental every hour, and a full load once a night or once a week. The full load fixes everything incremental missed: deletes and changes with no `updated_at` move.
- **Compare IDs.** Copy only the list of IDs (one small column) and delete warehouse rows whose ID is no longer in the source. Cheaper than a full load, but it only fixes deletes.
- **Change Data Capture (CDC).** Read changes from the PostgreSQL write-ahead log (WAL) through logical replication, for example with Debezium or AWS DMS (Database Migration Service). Every insert, update, and delete is captured, even manual SQL fixes. This is the most correct option, and also the most work to run. See [PostgreSQL TOAST and WAL](../database-patterns/postgres-toast-wal.md) for how the WAL works.

---

## Append-Only Tables

Some tables are only ever inserted into: logs, history tables, events. Rows are never updated or deleted. For these tables incremental load is a very good fit, and you can use the ID instead of a timestamp:

```sql
SELECT * FROM payment_history WHERE id > :last_max_id;
```

The same commit-order problem from step 4 applies here. Sequence values are given out when a row is inserted, not when the transaction commits, so ID 1001 can become visible before ID 1000. Keep an overlap (`id > :last_max_id - 1000`) and an idempotent load.

Before you rely on "append-only", check that it is really true. One admin screen that edits history rows breaks the whole idea.

---

## Which One to Use

| Situation | Choice |
|---|---|
| Small table (thousands of rows) | Full load. Simple and always correct. |
| Large table, rows are updated, no deletes | Incremental with trigger-managed `updated_at` |
| Large table with hard deletes | Incremental + periodic full reload, or CDC |
| Large append-only table | Incremental by ID |
| Data must be exact (finance, audit) | CDC, or incremental + a regular full reload to fix drift |

A good default for most teams: **incremental often, full load sometimes.** Incremental keeps the data fresh. The full load fixes what incremental missed.

And in both cases, read from a read replica, not the primary database. The load job then cannot slow down production writes.

---

### For AI agents

```
Copying data from PostgreSQL to a data warehouse (e.g. Redshift). Full load: copy the whole table each run and replace the warehouse copy. Always correct (picks up updates, deletes, manual SQL fixes), but slow and puts load on the source that grows with table size. Load into a staging table and swap with ALTER TABLE RENAME in one transaction (views on the table must be late-binding, WITH NO SCHEMA BINDING, or they keep pointing at the old table); never TRUNCATE + INSERT in Redshift (TRUNCATE commits immediately, readers see an empty table). Incremental load: copy only rows WHERE updated_at > watermark, then upsert. Fast, but misses rows changed without updated_at moving and never sees hard deletes - silently. Make it safe: set updated_at with a BEFORE UPDATE trigger, index updated_at (CREATE INDEX CONCURRENTLY), take the watermark from max(updated_at) loaded, not the job clock, re-read an overlap window (now() is transaction start time, so late commits can fall behind the watermark), and make the load idempotent (Redshift does not enforce primary keys - use MERGE or delete+insert from staging). Deletes: soft delete (deleted_at), periodic full reload, ID comparison, or CDC from the WAL (Debezium, AWS DMS). Append-only tables: incremental by id with an overlap. Default: incremental often + full reload sometimes. Read from a replica, not the primary.
```

Reference: `https://michalsniezko.github.io/database-patterns/full-vs-incremental-load.html`
