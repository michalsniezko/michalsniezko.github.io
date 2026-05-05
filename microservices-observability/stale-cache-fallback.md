---
layout: default
title: Stale Cache Fallback for Microservice Resilience
parent: Microservices & Service Design
nav_order: 7
---

## Stale Cache Fallback for Microservice Resilience

**Problem:** Your service fetches configuration from an upstream microservice on every request. This pattern is a natural complement to the [Repositories as Service Clients](repositories-service-clients.md) pattern, where the repository wraps an HTTP call that can fail. When that upstream is briefly unavailable, all downstream requests fail - even though the configuration data it serves changes at most a few times a day.

**Solution:** Implement a "stale-while-revalidate" cache. Store entries with a manually-tracked `expiresAt` timestamp but *without* setting a real TTL in APCu. This way expired entries stay in memory and can be served when the upstream is down. Normal reads respect the expiry; fallback reads ignore it.

### The Cache Layer

```php
class APCuCache implements CacheInterface
{
    public function __construct(private readonly int $ttlSeconds) {}

    public function set(string $key, mixed $value): bool
    {
        // No TTL passed to apcu_store - the entry never expires from APCu's perspective.
        // We manage expiry ourselves via the 'expiresAt' field.
        return apcu_store($key, [
            'value'     => $value,
            'expiresAt' => time() + $this->ttlSeconds,
        ]);
    }

    /** Returns the value only if not yet expired. Returns null otherwise. */
    public function get(string $key): mixed
    {
        $entry = apcu_fetch($key);
        if ($entry === false) {
            return null;
        }
        return $entry['expiresAt'] > time() ? $entry['value'] : null;
    }

    /** Returns the value regardless of expiry - used as a fallback during upstream outages. */
    public function getIgnoringExpiration(string $key): mixed
    {
        $entry = apcu_fetch($key);
        return $entry === false ? null : $entry['value'];
    }
}
```

The key decision: `apcu_store($key, $data)` is called without a third TTL argument. APCu will keep the entry indefinitely (until process restart or memory pressure). We track freshness ourselves.

### The Service

```php
class ParameterService
{
    public function __construct(
        private readonly APCuCache              $cache,
        private readonly UpstreamApiRepository $api,
        private readonly LoggerInterface       $logger,
    ) {}

    public function get(string $domain, string $key): mixed
    {
        $cacheKey = "{$domain}.{$key}";

        // 1. Happy path: fresh value in cache
        $cached = $this->cache->get($cacheKey);
        if ($cached !== null) {
            return $cached;
        }

        try {
            // 2. Cache miss or expired - fetch from upstream
            $value = $this->api->fetch($domain, $key);
            $this->cache->set($cacheKey, $value);
            return $value;

        } catch (\Throwable $e) {
            // 3. Upstream is down - try stale value
            $stale = $this->cache->getIgnoringExpiration($cacheKey);
            if ($stale !== null) {
                $this->logger->error('Serving stale cached value', [
                    'domain' => $domain,
                    'key'    => $key,
                    'error'  => $e->getMessage(),
                ]);
                return $stale;
            }

            // 4. No stale value either - nothing we can do
            throw new ParameterUnavailableException(
                "Cannot fetch '{$key}' for '{$domain}' and no cached value exists.",
                previous: $e
            );
        }
    }
}
```

### What Happens in Each Scenario

| Cache state | Upstream  | Result                                  |
|-------------|-----------|------------------------------------------|
| fresh       | up / down | Return cached value (no network call)   |
| expired     | up        | Fetch fresh, update cache, return fresh |
| expired     | down      | Return stale value + log error          |
| empty       | up        | Fetch fresh, populate cache, return     |
| empty       | down      | Throw ParameterUnavailableException     |

### Why Not Use APCu's Built-in TTL?

If you pass a TTL to `apcu_store`, APCu automatically evicts the entry when it expires. You can no longer fall back to it. By storing `expiresAt` yourself and calling `apcu_store` without a TTL, the entry stays resident and you can still serve it as a fallback - while still treating it as "stale" during normal operation.

The trade-off: stale entries accumulate in memory. This is acceptable when the key space is small (e.g. a few hundred configuration parameters). For large or unbounded key spaces, switch to a cache that supports explicit stale reads, like Redis with `OBJECT ENCODING` or a two-key pattern.

### Key Takeaways

| Design choice | Why |
|---|---|
| `apcu_store` without TTL | Entry survives expiry and can be served as stale fallback |
| Manual `expiresAt` field | You control freshness without losing the ability to read expired data |
| Two `get` methods | `get()` for normal reads; `getIgnoringExpiration()` only called on upstream failure |
| Log on stale serve | Alerts when the service is running on cached data - not silently degraded |

> **Note:** APCu is per-process, not shared across PHP-FPM workers. If you restart workers or deploy, the cache is cleared and the first request after restart will always hit the upstream. Plan accordingly: accept a cold-start upstream call or use a shared cache (Memcached, Redis) for the stale-fallback pattern.

---

### For AI agents

```
For resilience against upstream failures: cache responses without a hard TTL. Track expiry manually via cached_at. On cache hit + expired: attempt upstream call; on failure serve the stale entry. Always log when serving stale data so degradation is observable.
```

Reference: `https://michalsniezko.github.io/microservices-observability/stale-cache-fallback.html`
