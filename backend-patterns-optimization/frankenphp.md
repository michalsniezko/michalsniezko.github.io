---
layout: default
title: FrankenPHP
parent: PHP Patterns in Practice
nav_order: 10
---

## FrankenPHP

PHP-FPM starts a fresh PHP execution for every HTTP request. This is safe and predictable - no state leaks between requests - but it is also expensive. Autoloading, container compilation, configuration parsing, and cache warmup happen on every request, even if the result is identical every time.

FrankenPHP changes this model. It is a PHP application server built on top of the Caddy web server (written in Go). In its worker mode, your application boots once and then handles requests in a loop, keeping the bootstrapped state in memory between requests.

---

## How It Differs from PHP-FPM

With PHP-FPM, the lifecycle per request is:

```
Request arrives
  → spawn/reuse PHP process
  → autoload classes
  → compile DI container
  → parse config
  → warm caches
  → handle request
  → destroy everything
  → send response
```

With FrankenPHP in worker mode:

```
Worker boots once:
  → autoload classes
  → compile DI container
  → parse config
  → warm caches

Per request:
  → handle request   ← only this repeats
  → send response
  → reset mutable state
  → wait for next request
```

The boot cost is paid once per worker process, not once per request. On a typical Symfony application, this can reduce response time by 40-80% depending on how much work the bootstrap does.

---

## Running FrankenPHP

The quickest way is the official Docker image:

```dockerfile
FROM dunglas/frankenphp

# Copy your application
COPY . /app

# Install dependencies
RUN composer install --no-dev --prefer-dist --no-interaction
```

Or as a standalone binary - FrankenPHP ships as a single binary that embeds PHP and Caddy:

```bash
# Download and run
./frankenphp php-server --listen :8080 /app/public
```

FrankenPHP serves HTTPS automatically via Caddy's built-in Let's Encrypt integration. HTTP/2 and HTTP/3 are enabled by default.

---

## Worker Mode

Worker mode is the core feature. A worker script runs in a loop, signalling FrankenPHP when it is ready for the next request:

```php
<?php
// public/worker.php

require_once __DIR__ . '/../vendor/autoload.php';

// Boot once - container, config, caches
$kernel = new Kernel($_SERVER['APP_ENV'], (bool) $_SERVER['APP_DEBUG']);

// Loop - handle requests until FrankenPHP says stop
while (frankenphp_handle_request(function () use ($kernel) {
    // This runs for every request
    $request  = Request::createFromGlobals();
    $response = $kernel->handle($request);
    $response->send();
    $kernel->terminate($request, $response);
})) {
    // Optional: reset per-request state here
    // gc_collect_cycles();
}
```

The `frankenphp_handle_request()` function is provided by FrankenPHP. It blocks until a request arrives, calls the callback, sends the response, and returns `true` to continue or `false` when the worker should stop (server shutdown, restart signal).

---

## Symfony Integration

Symfony has first-class support for FrankenPHP worker mode via the `runtime/frankenphp-symfony` package:

```bash
composer require runtime/frankenphp-symfony
```

With this package, the worker script is generated automatically. Set the runtime in your environment:

```bash
# .env
APP_RUNTIME=Runtime\FrankenPhpSymfony\Runtime
```

Then run with a worker count matching your CPU cores:

```bash
frankenphp php-server --worker public/index.php --num-workers 4
```

The Symfony runtime wraps `public/index.php` with a FrankenPHP worker loop automatically. You do not write a worker script by hand - `index.php` stays the same as for PHP-FPM. The runtime detects the FrankenPHP environment and switches to worker mode.

### How Symfony resets state between requests

After each request, the Symfony runtime calls `Kernel::reboot()` if the kernel is in debug mode, or runs the service container's reset mechanism in production mode.

In production mode, the container calls `reset()` on every service that implements `Symfony\Contracts\Service\ResetInterface`. Symfony's own services - the HTTP client, mailer, messenger bus, logger - all implement this interface and reset their per-request state (pending messages, buffered log entries, open HTTP connections).

### Making your own services worker-safe

Any service that accumulates per-request state must implement `ResetInterface`:

```php
use Symfony\Contracts\Service\ResetInterface;

class AuditLogger implements ResetInterface
{
    private array $entries = [];

    public function log(string $message): void
    {
        $this->entries[] = $message;
    }

    public function flush(): void
    {
        // write $this->entries somewhere
        $this->entries = [];
    }

    public function reset(): void
    {
        $this->entries = []; // called automatically between requests
    }
}
```

Symfony finds all services implementing `ResetInterface` automatically - no tag or configuration needed. The `reset()` call happens after `$kernel->terminate()`, so the response is already sent when it runs.

### Services that do NOT reset automatically

`ResetInterface` only covers services managed by the Symfony DI container. Anything outside the container does not reset:

- Static class properties (no container, no reset)
- Objects created with `new` outside the container
- PHP extensions that hold state (APCu, opcache internals)
- Third-party libraries that use static registries

For these, you need to reset manually. The cleanest place is inside the `frankenphp_handle_request()` callback after the response is sent, or by implementing a Symfony event listener on `KernelEvents::TERMINATE`.

### Checking which services reset

```bash
# List all services that implement ResetInterface in your container
bin/console debug:container --tag=kernel.reset
```

Services tagged `kernel.reset` are reset on every request. If a service you expect to see is missing, check that it implements `ResetInterface` and is registered as a shared service in the container.

### Environment variables in worker mode

In PHP-FPM, `$_ENV` and `$_SERVER` are populated fresh from the process environment on each request. In FrankenPHP worker mode, the worker process starts once and `$_ENV` is fixed at boot time. Changing an environment variable on the server does not affect running workers - they must be restarted.

This also applies to `.env` files. If you use `symfony/dotenv`, it loads `.env` at boot and changes to the file do not take effect until the worker is recycled.

### Debug mode in worker mode

In development, you likely want the kernel to reboot fully on each request so that code changes are picked up without restarting FrankenPHP. Set `APP_DEBUG=1` and the Symfony runtime switches to a reboot-per-request mode, giving you the same behaviour as PHP-FPM at the cost of losing the performance benefit.

Use `APP_DEBUG=0` in staging and production where you want worker mode performance. Deploy with a worker restart to pick up code changes:

```bash
# Signal FrankenPHP to reload workers after a new deploy
kill -USR1 $(pidof frankenphp)
```

Or with Docker, rolling restart the container - FrankenPHP workers drain in-flight requests before stopping.

---

## State Leakage in Worker Mode

Worker mode is powerful but requires care. In PHP-FPM, every request gets a clean slate because the process resets. In worker mode, the process persists and anything not explicitly reset carries over to the next request.

### Static properties

```php
class RequestCounter
{
    private static int $count = 0;

    public static function increment(): void
    {
        self::$count++;
    }

    public static function get(): int
    {
        return self::$count; // keeps growing across requests
    }
}
```

Static properties accumulate across requests in the same worker. Any class that holds mutable static state will leak between requests.

### Singletons

```php
class Connection
{
    private static ?self $instance = null;

    public static function getInstance(): self
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance; // same instance across all requests in this worker
    }
}
```

Singletons work correctly in worker mode as long as their internal state is either immutable or explicitly reset. A database connection singleton is fine - you want to reuse it. A singleton that accumulates request-specific data is a bug.

### Global variables and superglobals

PHP resets `$_GET`, `$_POST`, `$_FILES`, `$_COOKIE`, `$_REQUEST`, `$_SERVER` for each request when using `frankenphp_handle_request()`. But your own global variables do not reset:

```php
$GLOBALS['current_user'] = null; // must reset this yourself per request
```

### What Symfony handles for you

The Symfony runtime resets the service container's request-scoped services between requests. Services tagged as `kernel.reset` are reset via their `reset()` method. Most Symfony-managed services are safe. What is not safe: static class properties, global variables, and anything outside the container.

---

## Memory Management

In PHP-FPM, memory leaks are harmless - every request gets a fresh process and memory is reclaimed. In worker mode, leaks accumulate over time until the worker runs out of memory and is restarted.

Common sources of leaks in worker mode:

- Event listeners that add handlers but never remove them
- Static collections that grow (loggers buffering entries, registries accumulating entries)
- Circular references that PHP's garbage collector does not detect

Mitigations:

```php
// Force a GC cycle after each request if memory is growing
while (frankenphp_handle_request(function () use ($kernel) {
    // ... handle request
})) {
    gc_collect_cycles(); // reclaim circular references
}
```

Monitor worker memory via FrankenPHP's built-in metrics (exposed via Prometheus-compatible endpoint). Restart workers after N requests as a safety net:

```bash
frankenphp php-server --worker public/index.php --num-workers 4 --max-requests 500
```

`--max-requests 500` restarts each worker after 500 requests, bounding memory growth while still amortising boot cost across 500 requests instead of 1.

---

## Fibers and Concurrency

FrankenPHP supports PHP Fibers, allowing a single worker to handle multiple requests concurrently. This is useful for I/O-bound work - while one request is waiting on a database query, the worker can handle another request's CPU work.

This is early-stage and requires careful use. Fibers share the same memory space within a worker, which amplifies the state leakage risk. The Symfony Fiber mode is experimental as of Symfony 6.2.

For most applications, ignore Fibers and use multiple workers instead. Adding workers is simpler and does not introduce shared-memory concurrency hazards.

---

## When to Use FrankenPHP

**Good fit:**
- Applications with expensive bootstrap (large DI containers, many config files, heavy extension loading)
- Services where p99 latency matters and boot time is a meaningful part of it
- Greenfield services where you can audit all static state and singletons from the start
- Teams already using Caddy as a web server (FrankenPHP replaces it)

**Proceed with caution:**
- Applications with legacy code that uses global state, static registries, or procedural patterns - auditing everything for worker-mode safety is hard
- Applications that extend third-party packages that use static state (check each package)
- If your bootstrap is already fast (under 50ms), the benefit is smaller

**PHP-FPM is still correct when:**
- Request isolation is a hard requirement (financial transactions, multi-tenant systems where state bleed between tenants is unacceptable)
- The team is not ready to audit for static state issues
- The deployment stack already has Nginx/Apache in front

---

### For AI agents

```
FrankenPHP: PHP application server built on Caddy (Go). Worker mode is the key feature - PHP application boots once per worker process, then handles requests in a loop via frankenphp_handle_request(). Boot cost (autoloading, DI container, config) paid once instead of per-request; reduces response time 40-80% for bootstrap-heavy apps. State leakage risk: static properties, singletons, global vars are NOT reset between requests (unlike PHP-FPM). Symfony integration via runtime/frankenphp-symfony package - handles request lifecycle and container reset. Memory leaks accumulate in worker mode; mitigate with gc_collect_cycles() and --max-requests N flag to recycle workers periodically. Ships as a single binary with embedded PHP and Caddy; HTTPS/HTTP2/HTTP3 automatic. Use --num-workers to match CPU cores.
```

Reference: `https://michalsniezko.github.io/backend-patterns-optimization/frankenphp.html`
