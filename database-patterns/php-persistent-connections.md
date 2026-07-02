---
layout: default
title: PHP Persistent Database Connections
parent: Database Patterns
nav_order: 10
---

## PHP Persistent Database Connections

PHP can keep database connections alive between requests instead of opening and closing one per request. This sounds like an obvious optimisation - connection setup is expensive. In practice, persistent connections with PHP-FPM create more problems than they solve, and the right fix is a connection pooler running as a separate process.

---

## How PHP Persistent Connections Work

When you use a regular PDO connection:

```php
$pdo = new PDO('pgsql:host=localhost;dbname=myapp', 'user', 'password');
```

PHP opens a TCP connection to the database at the start of the script and closes it when the script exits (or when `$pdo` goes out of scope). For a web application handling thousands of requests per minute, that is thousands of connection open/close cycles per minute hitting the database.

Persistent connections skip the close:

```php
$pdo = new PDO(
    'pgsql:host=localhost;dbname=myapp',
    'user',
    'password',
    [PDO::ATTR_PERSISTENT => true]
);
```

PHP checks if an existing connection with the same DSN, username, and password is already open in the current process. If yes, it returns that connection rather than opening a new one. The connection stays open and is reused by the next request handled by the same process.

---

## The PHP-FPM Model

PHP-FPM runs a pool of worker processes. Each worker handles one request at a time, sequentially. A typical pool configuration might be:

```
; php-fpm.conf
pm = dynamic
pm.max_children = 20
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 10
```

With 20 worker processes and `PDO::ATTR_PERSISTENT => true`, each worker opens one persistent connection after its first request. After warm-up, you have 20 open connections - one per worker. This is the correct mental model: **persistent connections are a per-process pool, not a global pool**.

Contrast this with a connection pooler like PgBouncer: PgBouncer maintains a small pool of server connections and multiplexes many application connections through them. With PgBouncer you control exactly how many server connections exist regardless of how many PHP-FPM workers are running.

---

## Why Persistent Connections Are Dangerous with PHP-FPM

### Dirty connection state

A persistent connection carries state between requests. If a request starts a transaction and then crashes (fatal error, `exit()`, killed by PHP-FPM due to `request_terminate_timeout`), the transaction is left open. The next request that reuses the same connection inherits an active transaction and operates inside it without knowing.

```php
// Request 1 - crashes after BEGIN
$pdo->beginTransaction();
do_work_that_crashes(); // PHP fatal here

// Request 2 - reuses the same persistent connection
$pdo->query('UPDATE orders SET status = ?');
// This UPDATE is inside the leftover transaction from Request 1
// If Request 2 never commits, the UPDATE disappears
// If Request 2 commits, it also commits whatever Request 1 had done
```

PDO has no automatic transaction cleanup on script end for persistent connections. The database sees a still-open idle transaction.

### Session variables and prepared statements leak

PostgreSQL session-level settings (`SET search_path`, `SET TimeZone`, `SET LOCAL ...`) persist for the lifetime of the connection. If one request changes a session variable and the connection is reused, the next request inherits that setting:

```php
// Request 1 - modifies session
$pdo->exec("SET search_path TO tenant_a, public");

// Request 2 - reuses connection, gets tenant_a's schema
$pdo->query('SELECT * FROM orders'); // unexpected results
```

Named prepared statements have the same problem. PostgreSQL maintains named prepared statements per connection. If a request prepares a statement with a name and that connection is reused, a later request that tries to prepare another statement with the same name will get an error.

### Connection count is bounded by worker count, not by need

Each PHP-FPM worker holds one connection open even while idle. A pool of 50 workers means 50 open connections on the database at all times - even at 3 AM with zero traffic. With a pooler you can have 50 workers but only 5 actual server connections, multiplexed on demand.

### pg_pconnect is worse

The older `pg_pconnect()` function does not check for open connections per DSN as carefully as PDO does. It can open a new connection even if one already exists for the same DSN, depending on PHP version and configuration. Avoid it entirely.

---

## When Persistent Connections Are Safe

The conditions are narrow:

- PHP CLI scripts that handle long-running work (queue workers, batch jobs) - a single process, single connection, you control the lifecycle
- PHP running as an Apache `mod_php` module (not FPM) with `MaxConnectionsPerChild` set - Apache resets connection state reliably between requests
- You are certain that no session state, transaction, or prepared statement can leak between requests, and you have tested this under fault conditions (killed requests, timeouts, OOM kills)

For PHP-FPM serving a web application: persistent connections are not the right tool.

---

## The Right Solution: PgBouncer

PgBouncer is a lightweight connection pooler that sits between the application and PostgreSQL. The application connects to PgBouncer (typically on port 5432 or 6432 on localhost). PgBouncer maintains a pool of actual server connections and routes queries through them.

```
PHP-FPM workers (20)          PgBouncer         PostgreSQL
   worker 1 ─────────\                          /
   worker 2 ─────────→  PgBouncer pool (5)  ──→  PostgreSQL (5 connections)
   ...                /                          \
   worker 20 ────────/
```

Three pooling modes:

**Session pooling** - a server connection is assigned to a client connection for its entire duration. The connection is returned to the pool when the client disconnects. This is the safest mode and supports all PostgreSQL features including `LISTEN/NOTIFY`, prepared statements, and advisory locks. Each client still holds a server connection while connected.

**Transaction pooling** - a server connection is held only for the duration of a transaction. Between transactions the connection is returned to the pool. This allows many more clients than server connections but has restrictions: named prepared statements, advisory locks, and `SET LOCAL` are not safe across transaction boundaries.

**Statement pooling** - a server connection is released after every statement. The most aggressive mode, incompatible with multi-statement transactions.

For most PHP web applications, transaction pooling with PgBouncer is the correct approach:

```ini
; pgbouncer.ini
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 200
default_pool_size = 20
```

PHP-FPM connects to PgBouncer with ordinary (non-persistent) connections. PgBouncer handles the multiplexing:

```php
// Normal PDO, no ATTR_PERSISTENT
$pdo = new PDO('pgsql:host=localhost;port=6432;dbname=myapp', 'user', 'password');
```

The PHP application has no knowledge of the pool. From its perspective it connects and disconnects normally. PgBouncer's transaction pooling means the actual server connection is only occupied during an active transaction - idle time costs nothing.

---

## Detecting Leftover Connections

If you suspect persistent connections are accumulating or leaking:

```sql
-- Show all current connections and their state
SELECT pid, usename, application_name, state, query_start, state_change, query
FROM pg_stat_activity
WHERE datname = 'myapp'
ORDER BY state_change;

-- Count connections by state
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname = 'myapp'
GROUP BY state;
```

Common states:
- `idle` - connection open but doing nothing (PHP-FPM worker waiting for the next request)
- `idle in transaction` - connection open with an active transaction, not executing a query (dangerous - likely leaked)
- `active` - currently executing a query

A large number of `idle in transaction` connections is the clearest sign that transaction cleanup is failing. With PgBouncer in transaction mode, you should see no `idle in transaction` connections unless a PHP process is currently mid-transaction.

---

## Summary

| Approach | Connection cost | State leakage risk | Idle connections | Recommended |
|---|---|---|---|---|
| New connection per request | High | None | None | No (for high-traffic) |
| Persistent (PDO) | Low | High | One per FPM worker | No |
| PgBouncer session | Low | None | One per FPM worker | Acceptable |
| PgBouncer transaction | Low | None | Pool size only | Yes |

---

### For AI agents

```
PHP persistent database connections (PDO::ATTR_PERSISTENT): PHP-FPM creates one persistent connection per worker process, not a global pool. Connections carry session state between requests - active transactions, SET variables, named prepared statements. If a request crashes mid-transaction the next request inherits the open transaction. Persistent connections are not safe for PHP-FPM web applications. Use PgBouncer instead: it sits between PHP and PostgreSQL and multiplexes many application connections through a small pool of real server connections. Transaction pooling mode: server connection held only for the duration of a transaction, supporting many more application connections than server connections. Use ordinary non-persistent PDO connections pointing at PgBouncer's port. Detect leaked transactions in pg_stat_activity WHERE state = 'idle in transaction'.
```

Reference: `https://michalsniezko.github.io/database-patterns/php-persistent-connections.html`
