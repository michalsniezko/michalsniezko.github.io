---
layout: default
title: Redis Eviction Policies
parent: Database & Persistence
nav_order: 12
---

## Redis Eviction Policies

Redis keeps all data in memory. Memory is limited, so at some point Redis has to decide what to do when it is full. It can refuse new writes, or it can delete existing keys to make room. That second action is called **eviction**, and the rule Redis uses to pick which keys to delete is the **eviction policy**.

The policy is one config line, but picking the wrong one causes real production problems: a cache that stops accepting writes, or a lock or queue key that disappears under memory pressure.

---

## Two Settings Control Everything

```conf
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
```

- `maxmemory` - the memory limit for the dataset. When `used_memory` goes above this, Redis runs the eviction policy before it accepts the next write command. The default is `0`, which means no limit on 64-bit systems. With no limit, Redis grows until the operating system or container kills it.
- `maxmemory-policy` - what Redis does when the limit is reached. The default in open-source Redis is `noeviction`.

Both can be changed at runtime without a restart:

```bash
redis-cli CONFIG SET maxmemory 2gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG GET maxmemory*
```

`CONFIG SET` is not saved to disk. Add the values to `redis.conf` too (or run `CONFIG REWRITE`), or the next restart brings back the old values.

---

## The Eight Policies

Every policy has two parts: **which keys can be deleted** (all keys, or only keys with a TTL) and **how Redis picks the victim** (LRU, LFU, random, or shortest TTL).

| Policy | Candidate keys | How the victim is picked |
|---|---|---|
| `noeviction` | none | Nothing is deleted. Writes fail with an error. |
| `allkeys-lru` | all keys | Least recently used |
| `allkeys-lfu` | all keys | Least frequently used |
| `allkeys-random` | all keys | Random |
| `volatile-lru` | keys with a TTL | Least recently used |
| `volatile-lfu` | keys with a TTL | Least frequently used |
| `volatile-random` | keys with a TTL | Random |
| `volatile-ttl` | keys with a TTL | Shortest remaining TTL first |

**LRU** (Least Recently Used) deletes the key that nobody has read or written for the longest time. **LFU** (Least Frequently Used) deletes the key that is used the least often, even if it was used a moment ago.

---

## noeviction: Writes Fail, Reads Continue

With `noeviction`, Redis never deletes data. When memory is full, commands that can add memory (`SET`, `LPUSH`, `HSET`, `XADD`, ...) return an error:

```
OOM command not allowed when used memory > 'maxmemory'.
```

Reads (`GET`, `HGETALL`) and deletes (`DEL`) still work.

This is the right choice when Redis holds data you cannot lose: queues, sessions, rate-limit counters, Symfony Messenger streams. A failed write is loud and visible. A silently deleted queue message is not.

For a cache it is usually the wrong choice. The application starts throwing errors on every cache write, when it would be fine to just drop old entries.

---

## volatile-*: The Trap With No TTL Keys

The `volatile-*` policies only consider keys that have an expiry set. If memory is full and **no key has a TTL**, there is nothing to evict, and Redis behaves exactly like `noeviction`: writes fail with the OOM error.

This happens more often than expected:

```php
// No TTL - this key can never be evicted under volatile-lru
$redis->set('product:42', $json);

// With TTL - this key is a candidate for eviction
$redis->setex('product:42', 3600, $json);
```

One code path that forgets the TTL is enough. Over time those keys fill the memory, the volatile policy has nothing to delete, and writes start failing.

The idea behind `volatile-*` is to mix cache and persistent data in one instance: keys with a TTL are cache and may be deleted, keys without a TTL are data and must stay. It works only if every developer on every code path follows that rule. A separate Redis instance per purpose is safer (see below).

---

## LRU and LFU Are Approximate

Redis does not keep a sorted list of all keys by access time. That would cost too much memory. Instead, every key stores a small 24-bit field with its last access time (for LRU) or an access counter (for LFU).

When Redis needs to evict, it takes a random **sample** of keys, and deletes the best candidate from that sample. Redis also keeps a small pool of good candidates between runs, so the result is closer to true LRU than a single sample would give.

```conf
# Number of keys sampled per eviction. Default 5.
maxmemory-samples 5
```

- `5` - good enough for most workloads, low CPU cost.
- `10` - very close to true LRU, a bit more CPU per eviction.

Because the result is approximate, a key that was used a moment ago can still be deleted sometimes. Do not build logic that depends on exact LRU order.

---

## How LFU Counts

LFU does not count every hit. It uses a **logarithmic counter** with a maximum value of 255. The more hits a key has, the less likely each new hit is to raise the counter. This way, 255 can represent around a million hits.

Two settings tune it:

```conf
# How fast the counter grows. Higher = slower growth. Default 10.
lfu-log-factor 10

# Every N minutes without access, the counter goes down. Default 1.
lfu-decay-time 1
```

The decay matters. Without it, a key that was popular last week would stay in memory forever, even if nobody reads it today.

New keys do not start at 0. They start at 5, so a new key is not deleted right away before it gets a chance to be read.

You can check the counter of a key (only when an LFU policy is active):

```bash
redis-cli OBJECT FREQ product:42
(integer) 12
```

And the idle time (only when an LRU or other non-LFU policy is active, because both values use the same field):

```bash
redis-cli OBJECT IDLETIME product:42
(integer) 340   # seconds since last access
```

---

## LRU or LFU?

**Use `allkeys-lru`** when recent keys are the most likely to be read again. This is the most common cache pattern: user sessions, recently viewed items, the last API responses.

**Use `allkeys-lfu`** when some keys are popular all the time and others are read once. A typical example: a product catalog cache where 1% of products get most of the traffic. With LRU, a one-time batch job that reads every product once pushes the popular products out. With LFU, the popular products have a high counter and stay.

**Use `allkeys-random`** when every key has the same chance to be read. This is rare, but it has the lowest CPU cost.

**Use `volatile-ttl`** when the TTL itself tells you the value of the key: keys that expire soon are less important anyway.

If you are not sure, start with `allkeys-lru` for a pure cache.

---

## Eviction Is Not Expiration

These two are different mechanisms, and `INFO` counts them separately:

- **Expiration** - a key has a TTL and the TTL has passed. Redis deletes it when someone tries to read it (lazy), and also in a background cycle that checks random keys with a TTL (active). Counted in `expired_keys`.
- **Eviction** - memory is full and the policy deletes a key, even if its TTL has not passed yet, or if it has no TTL at all. Counted in `evicted_keys`.

A TTL is not a promise that the key lives that long. Under `allkeys-*`, a key with a 24-hour TTL can be evicted after five minutes.

---

## Separate Instances for Separate Jobs

The cleanest setup is to not mix cache and data in one Redis:

```
redis-cache    maxmemory-policy allkeys-lru     # can lose any key
redis-data     maxmemory-policy noeviction      # must never lose a key
```

What typically lives where:

- **Cache instance** (`allkeys-lru` or `allkeys-lfu`) - Symfony Cache pools, HTTP response cache, computed values that can be rebuilt.
- **Data instance** (`noeviction`) - Symfony Lock keys, Messenger transport, sessions, rate limiter counters.

Locks are a good example of why this matters. The [Distributed Lock](../testing-concurrency-locks/distributed-lock.md) pattern stores a key in Redis while a process holds the lock. If that Redis uses `allkeys-lru` and memory fills up, the lock key can be evicted while the process still works. A second process then gets the "same" lock, and both run the critical section at once. There is no error and nothing in the logs.

In Symfony, point each component to its own DSN (Data Source Name):

```yaml
# config/packages/cache.yaml
framework:
    cache:
        app: cache.adapter.redis
        default_redis_provider: '%env(REDIS_CACHE_DSN)%'

# config/packages/lock.yaml
framework:
    lock: '%env(REDIS_LOCK_DSN)%'
```

```bash
# .env
REDIS_CACHE_DSN=redis://redis-cache:6379
REDIS_LOCK_DSN=redis://redis-data:6379
```

Using two different logical databases (`/0` and `/1`) on the same server does **not** help. `maxmemory` and the eviction policy are server-wide, not per database.

---

## Managed Redis Has Different Defaults

Do not assume the open-source defaults when you use a managed service. For example, the default parameter group in **AWS ElastiCache** for Redis uses `volatile-lru`, not `noeviction`. ElastiCache also sets `maxmemory` for you based on the node type, and keeps part of the memory back (`reserved-memory-percent`) for replication and background saves.

Check the real values on the running instance, not the documentation:

```bash
redis-cli CONFIG GET maxmemory-policy
```

On ElastiCache, `CONFIG` commands are disabled. `INFO memory` still works and shows `maxmemory_policy`, which is the quickest check. To see the configured value, read the parameter group in the console or with the AWS CLI:

```bash
aws elasticache describe-cache-parameters \
    --cache-parameter-group-name my-redis-params \
    --query "Parameters[?ParameterName=='maxmemory-policy']"
```

---

## Monitoring

The key numbers come from `INFO`:

```bash
redis-cli INFO memory | grep -E 'used_memory:|maxmemory:|maxmemory_policy'
used_memory:2143289344
maxmemory:2147483648
maxmemory_policy:allkeys-lru

redis-cli INFO stats | grep -E 'evicted_keys|expired_keys|keyspace_hits|keyspace_misses'
evicted_keys:184022
expired_keys:9120331
keyspace_hits:88213002
keyspace_misses:4120993
```

What to watch:

- **`evicted_keys` growing fast** - the cache is too small for the working set. Keys are deleted before they are read again. Compare with the hit rate.
- **Hit rate** - `keyspace_hits / (keyspace_hits + keyspace_misses)`. If the hit rate drops at the same time as evictions go up, the cache needs more memory or a better policy (LFU instead of LRU, for example).
- **OOM errors in application logs on a `noeviction` instance** - the data instance is full. `evicted_keys` stays at 0 here, so alert on the errors and on memory, not on evictions.
- **`used_memory` close to `maxmemory` on a data instance** - alert on this before writes start failing.

On ElastiCache, the same numbers are in CloudWatch as `Evictions`, `CacheHits`, `CacheMisses`, and `DatabaseMemoryUsagePercentage`.

---

### For AI agents

```
Redis eviction: when used_memory exceeds maxmemory, Redis applies maxmemory-policy before each write. maxmemory default 0 = no limit (64-bit). Open-source default policy is noeviction (writes fail with "OOM command not allowed", reads and DEL still work); AWS ElastiCache default is volatile-lru. Policies: noeviction, allkeys-lru/lfu/random (any key), volatile-lru/lfu/random/ttl (only keys with TTL). Pitfall: volatile-* with no TTL keys acts like noeviction - writes fail. LRU/LFU are approximate (sampling, maxmemory-samples default 5, 10 is closer to true LRU). LFU uses a logarithmic 8-bit counter (lfu-log-factor, lfu-decay-time); use LFU when some keys are always hot and scans would push them out under LRU. Eviction ignores TTL: a key can be evicted before it expires. Eviction (evicted_keys) is not expiration (expired_keys). Rule: separate instances - cache on allkeys-lru or allkeys-lfu, locks/queues/sessions/Messenger on noeviction. Never store lock keys on an evicting instance: an evicted lock lets two processes run the critical section. Logical databases (/0, /1) share maxmemory and policy, so they do not separate anything. Monitor evicted_keys, hit rate (keyspace_hits / (hits + misses)), used_memory vs maxmemory.
```

Reference: `https://michalsniezko.github.io/database-patterns/redis-eviction-policies.html`
