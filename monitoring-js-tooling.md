---
layout: default
title: Monitoring with TICK Stack, JavaScript Tooling
nav_order: 8
---

# Monitoring with TICK Stack & JavaScript Tooling

## The TICK Stack

**Use Case:** You need a self-hosted monitoring pipeline that collects metrics from 50+ services, stores time-series data efficiently, visualizes it in dashboards, and alerts your team when things break — without paying per-host SaaS fees.

TICK is four components, each replaceable independently:

```
Telegraf ──► InfluxDB ──► Chronograf
  (collect)    (store)      (visualize)
                 │
             Kapacitor
              (alert)
```

| Component    | Role                          | Analogy                        |
|-------------|-------------------------------|--------------------------------|
| **Telegraf**  | Agent that collects metrics (CPU, RAM, disk, custom)  | Prometheus node_exporter  |
| **InfluxDB**  | Time-series database optimized for write-heavy workloads | Prometheus TSDB          |
| **Chronograf**| Dashboard UI for queries and visualization              | Grafana                  |
| **Kapacitor** | Stream processing engine for alerting rules             | Prometheus Alertmanager  |

### Telegraf Agent Config

```toml
# /etc/telegraf/telegraf.conf

[agent]
  interval = "10s"
  flush_interval = "10s"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "${INFLUX_TOKEN}"
  organization = "engineering"
  bucket = "system-metrics"

# System metrics
[[inputs.cpu]]
  percpu = false
  totalcpu = true

[[inputs.mem]]

[[inputs.disk]]
  ignore_fs = ["tmpfs", "devtmpfs"]

# SQS queue depth (custom)
[[inputs.cloudwatch]]
  region = "eu-west-1"
  namespace = "AWS/SQS"
  period = "1m"
  [inputs.cloudwatch.metrics]
    names = ["ApproximateNumberOfMessagesVisible"]
    [[inputs.cloudwatch.metrics.dimensions]]
      name = "QueueName"
      value = "order-processing"
```

### Docker Compose (Full Stack)

```yaml
services:
  telegraf:
    image: telegraf:1.30
    volumes:
      - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
    depends_on: [influxdb]

  influxdb:
    image: influxdb:2.7
    ports: ["8086:8086"]
    volumes:
      - influx-data:/var/lib/influxdb2
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: changeme123
      DOCKER_INFLUXDB_INIT_ORG: engineering
      DOCKER_INFLUXDB_INIT_BUCKET: system-metrics

  chronograf:
    image: chronograf:1.10
    ports: ["8888:8888"]
    depends_on: [influxdb]

  kapacitor:
    image: kapacitor:1.7
    depends_on: [influxdb]
    volumes:
      - ./kapacitor.conf:/etc/kapacitor/kapacitor.conf:ro
      - ./tickscripts:/var/lib/kapacitor/tickscripts

volumes:
  influx-data:
```

> **Clean Code Tip:** Telegraf has 300+ input plugins. Only enable what you need. Each plugin adds collection overhead, network traffic, and storage cost. Start with `cpu`, `mem`, `disk`, and `net` — add plugins when a specific question demands the data, not preemptively.

---

## InfluxDB for Metrics

**Use Case:** Your PostgreSQL database stores 30 days of CPU metrics for 200 servers. The table has 500M rows, queries take 40 seconds, and storage eats 80GB. A time-series DB handles the same data in ~5GB with sub-second queries because it's purpose-built for timestamped, append-heavy, read-by-range workloads.

### Why Not Postgres?

| Concern          | Relational DB                       | InfluxDB                             |
|------------------|-------------------------------------|--------------------------------------|
| Write pattern    | Row-level locks, WAL overhead       | Append-only, batch-optimized         |
| Compression      | Generic (TOAST)                     | Column-oriented, delta/RLE encoding  |
| Retention        | Manual `DELETE WHERE ts < ...`      | Built-in retention policies          |
| Downsampling     | Materialized views (manual)         | Continuous queries / tasks           |
| Query language   | SQL (verbose for time ranges)       | Flux / InfluxQL (time-native)        |

### Flux Query: Average CPU Over 5-Minute Windows

```flux
from(bucket: "system-metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "cpu" and r._field == "usage_idle")
  |> aggregateWindow(every: 5m, fn: mean)
  |> map(fn: (r) => ({r with _value: 100.0 - r._value}))  // convert idle → usage
  |> yield(name: "cpu_usage")
```

### Retention Policy (Auto-Delete Old Data)

```bash
influx bucket update \
    --id <bucket-id> \
    --retention 30d   # auto-delete data older than 30 days
```

### Downsampling Task (Keep Hourly Averages Long-Term)

```flux
option task = {name: "downsample-cpu", every: 1h}

from(bucket: "system-metrics")
  |> range(start: -task.every)
  |> filter(fn: (r) => r._measurement == "cpu")
  |> aggregateWindow(every: 1h, fn: mean)
  |> to(bucket: "system-metrics-longterm")  // 1-year retention
```

> **Clean Code Tip:** Design two buckets from day one: a high-resolution bucket (10s intervals, 30-day retention) and a downsampled bucket (1h intervals, 1-year retention). This pattern keeps dashboards fast for recent data while preserving historical trends at negligible storage cost. Retrofitting downsampling onto a bucket with 6 months of raw data is painful.

---

## Kapacitor & Telegram Alerts

**Use Case:** Email alerts get buried. Slack channels get muted. A Telegram bot message pings your phone directly with a sound — hard to ignore at 3 AM when your order queue is backed up.

Kapacitor processes streams from InfluxDB and evaluates TICKscript rules in real time. When a condition is met, it fires an alert to any configured handler (Telegram, Slack, PagerDuty, webhook).

### Kapacitor Config (Telegram Handler)

```toml
# kapacitor.conf
[telegram]
  enabled = true
  url = "https://api.telegram.org/bot"
  token = "${TELEGRAM_BOT_TOKEN}"
  chat-id = "-1001234567890"
  parse-mode = "HTML"
  disable-web-page-preview = true
  disable-notification = false
```

### TICKscript: CPU Alert

```tickscript
// cpu_alert.tick
stream
    |from()
        .measurement('cpu')
        .where(lambda: "cpu" == 'cpu-total')
    |window()
        .period(5m)
        .every(1m)
    |mean('usage_idle')
    |eval(lambda: 100.0 - "mean")
        .as('usage_percent')
    |alert()
        .warn(lambda: "usage_percent" > 70)
        .crit(lambda: "usage_percent" > 90)
        .stateChangesOnly()           // don't spam on sustained high CPU
        .message('
<b>{{ .Level }}: CPU Alert</b>
Host: {{ index .Tags "host" }}
CPU Usage: {{ index .Fields "usage_percent" | printf "%.1f" }}%
Time: {{ .Time }}
        ')
        .telegram()
```

### TICKscript: SQS Queue Depth

```tickscript
// queue_alert.tick
stream
    |from()
        .measurement('cloudwatch_aws_sqs')
        .where(lambda: "queue_name" == 'order-processing')
    |window()
        .period(3m)
        .every(1m)
    |mean('approximate_number_of_messages_visible')
    |alert()
        .warn(lambda: "mean" > 1000)
        .crit(lambda: "mean" > 5000)
        .stateChangesOnly()
        .message('
<b>{{ .Level }}: SQS Backlog</b>
Queue: order-processing
Messages: {{ index .Fields "mean" | printf "%.0f" }}
        ')
        .telegram()
```

### Loading TICKscripts

```bash
kapacitor define cpu_alert -type stream -tick cpu_alert.tick \
    -dbrp "system-metrics.autogen"
kapacitor enable cpu_alert
kapacitor show cpu_alert   # verify status
```

> **Clean Code Tip:** `.stateChangesOnly()` is the single most important line in any TICKscript. Without it, Kapacitor fires an alert every evaluation cycle while the condition persists — your Telegram bot sends 60 messages per hour for sustained high CPU. State-change alerts fire once on breach and once on recovery. Add `.flapping(0.2, 0.5)` if alerts toggle between OK and WARN rapidly.

---

## Rollup.js Bundler

**Use Case:** You're building a shared JS/TS library consumed by multiple frontend apps. Webpack bundles everything into one fat file (good for apps). Rollup produces clean ES modules with tree-shaking that lets the consuming app drop unused exports (good for libraries).

### Why Rollup Over Webpack for Libraries

| Concern               | Webpack                            | Rollup                              |
|-----------------------|------------------------------------|--------------------------------------|
| Output format         | Proprietary module wrapper         | Clean ES modules / CommonJS          |
| Tree-shaking          | Works, but less aggressive         | Best-in-class — dead code elimination |
| Code splitting        | Excellent for apps                 | Limited (app bundling not its strength) |
| Config complexity     | Loaders, plugins, dev server       | Minimal for library use cases        |
| Bundle size (library) | Larger due to runtime overhead     | Smaller, no runtime wrapper          |

### `rollup.config.js` for a Shared Library

```javascript
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import typescript from '@rollup/plugin-typescript';
import { terser } from 'rollup-plugin-terser';

export default {
    input: 'src/index.ts',

    // Peer dependencies — DO NOT bundle these
    external: [
        'react',
        'react-dom',
        'axios',
        /^@company\//,  // regex: exclude all internal packages
    ],

    output: [
        {
            file: 'dist/index.cjs.js',
            format: 'cjs',           // CommonJS for Node/legacy
            sourcemap: true,
        },
        {
            file: 'dist/index.esm.js',
            format: 'esm',           // ES Modules for modern bundlers
            sourcemap: true,
        },
    ],

    plugins: [
        resolve(),                    // resolve node_modules
        commonjs(),                   // convert CJS deps to ESM
        typescript({ tsconfig: './tsconfig.build.json' }),
        terser({ format: { comments: false } }),
    ],
};
```

### `package.json` Entry Points

```json
{
    "name": "@company/ui-utils",
    "version": "2.1.0",
    "main": "dist/index.cjs.js",
    "module": "dist/index.esm.js",
    "types": "dist/index.d.ts",
    "files": ["dist"],
    "peerDependencies": {
        "react": ">=18.0.0",
        "axios": ">=1.0.0"
    },
    "scripts": {
        "build": "rollup -c",
        "build:watch": "rollup -c --watch"
    }
}
```

### Tree-Shaking in Action

```typescript
// src/index.ts — library exports 10 utilities
export { formatCurrency } from './formatters/currency';
export { formatDate } from './formatters/date';
export { debounce } from './utils/debounce';
export { throttle } from './utils/throttle';
// ... 6 more exports

// Consumer app — only imports 2
import { formatCurrency, debounce } from '@company/ui-utils';
// Rollup's ESM output lets the consumer's bundler drop the other 8 completely
```

> **Clean Code Tip:** The `external` array is the most commonly misconfigured option. If you forget to exclude `react`, Rollup bundles React into your library — every app that imports your library ships two copies of React (148KB duplicated). Rule: everything in `peerDependencies` goes in `external`. Use a regex pattern for internal packages to avoid maintaining a long list.

---

## Wrap Interceptor Pattern (Node.js Lambda)

**Use Case:** Every Lambda handler needs the same boilerplate: parse the event, log the request, catch errors, format the response, log the duration. Copy-pasting this into 30 handlers means 30 places to update when you change the error format or add a new header.

**Solution:** A higher-order function that wraps any handler with cross-cutting concerns. The business logic handler stays pure — it receives parsed input and returns data. The wrapper handles HTTP plumbing.

### The Interceptor

```javascript
// lib/interceptor.js

function wrapHandler(businessLogic, options = {}) {
    const { requireAuth = true } = options;

    return async (event, context) => {
        const start = Date.now();
        const requestId = context.awsRequestId;

        console.log(JSON.stringify({
            event: 'request_received',
            requestId,
            path: event.path,
            method: event.httpMethod,
            // Never log event.body in production — may contain PII
        }));

        try {
            // Auth check (if required)
            if (requireAuth && !event.headers?.Authorization) {
                return response(401, { error: 'Missing authorization header' });
            }

            // Parse body safely
            const body = event.body ? JSON.parse(event.body) : null;

            // Call the actual business logic
            const result = await businessLogic({ body, event, context });

            console.log(JSON.stringify({
                event: 'request_completed',
                requestId,
                durationMs: Date.now() - start,
                statusCode: 200,
            }));

            return response(200, result);

        } catch (error) {
            console.error(JSON.stringify({
                event: 'request_failed',
                requestId,
                durationMs: Date.now() - start,
                error: error.message,
                stack: error.stack,
            }));

            if (error.name === 'ValidationError') {
                return response(400, { error: error.message });
            }

            return response(500, { error: 'Internal server error' });
        }
    };
}

function response(statusCode, body) {
    return {
        statusCode,
        headers: {
            'Content-Type': 'application/json',
            'X-Request-Id': body?.requestId || 'unknown',
        },
        body: JSON.stringify(body),
    };
}

module.exports = { wrapHandler };
```

### Clean Business Logic Handlers

```javascript
// handlers/createOrder.js
const { wrapHandler } = require('../lib/interceptor');
const { OrderService } = require('../core/orderService');

const service = new OrderService();  // reused across warm invocations

module.exports.handler = wrapHandler(async ({ body }) => {
    // Pure business logic — no HTTP plumbing
    const order = await service.create({
        productId: body.productId,
        quantity: body.quantity,
    });

    return { orderId: order.id, status: order.status };
});

// handlers/healthCheck.js
const { wrapHandler } = require('../lib/interceptor');

module.exports.handler = wrapHandler(
    async () => ({ status: 'ok', timestamp: new Date().toISOString() }),
    { requireAuth: false }
);
```

> **Clean Code Tip:** Resist the urge to make the interceptor configurable for every edge case (custom serializers, per-route middleware chains, plugin systems). That's an HTTP framework — use Express or Fastify at that point. The wrapper should handle 3 things: logging, error formatting, and response shaping. Anything more complex means your Lambda is doing too much.

---

## Environment Syncing: Frontend Dictionaries from Backend DTOs

**Use Case:** The backend serves a `/api/v1/filters/dictionaries` endpoint (covered in [backend-patterns-optimization.md](./backend-patterns-optimization.md)) that returns all possible filter values. The frontend needs to consume this and keep its TypeScript types in sync — otherwise the backend adds a new fuel type and the frontend's type system doesn't know it exists.

### Backend Response (Recap)

```json
{
    "makes": [
        {"key": "bmw", "label": "BMW", "group": "German"},
        {"key": "toyota", "label": "Toyota", "group": "Japanese"}
    ],
    "fuelTypes": [
        {"key": "petrol", "label": "Petrol"},
        {"key": "diesel", "label": "Diesel"},
        {"key": "electric", "label": "Electric"}
    ]
}
```

### TypeScript Types (Mirroring the DTO)

```typescript
// types/dictionary.ts

interface DictionaryEntry {
    key: string;
    label: string;
    group?: string;
}

interface FilterDictionary {
    makes: DictionaryEntry[];
    fuelTypes: DictionaryEntry[];
    colors: DictionaryEntry[];
    transmissions: DictionaryEntry[];
}
```

### Fetching and Caching

```typescript
// services/dictionaryService.ts

let cachedDictionary: FilterDictionary | null = null;
let cacheExpiry = 0;

const CACHE_TTL_MS = 5 * 60 * 1000; // 5 minutes

export async function getDictionary(): Promise<FilterDictionary> {
    if (cachedDictionary && Date.now() < cacheExpiry) {
        return cachedDictionary;
    }

    const response = await fetch('/api/v1/filters/dictionaries');

    if (!response.ok) {
        if (cachedDictionary) {
            console.warn('Dictionary fetch failed, using stale cache');
            return cachedDictionary;  // stale is better than broken
        }
        throw new Error(`Dictionary fetch failed: ${response.status}`);
    }

    cachedDictionary = await response.json();
    cacheExpiry = Date.now() + CACHE_TTL_MS;

    return cachedDictionary!;
}
```

### Usage in a Filter Component

```typescript
// components/VehicleFilter.tsx

import { useEffect, useState } from 'react';
import { getDictionary, type FilterDictionary, type DictionaryEntry } from '../services/dictionaryService';

function VehicleFilter({ onFilterChange }) {
    const [dictionary, setDictionary] = useState<FilterDictionary | null>(null);

    useEffect(() => {
        getDictionary().then(setDictionary);
    }, []);

    if (!dictionary) return <div>Loading filters...</div>;

    return (
        <div>
            <SelectFilter
                label="Make"
                options={dictionary.makes}
                grouped={true}
                onChange={(key) => onFilterChange('make', key)}
            />
            <SelectFilter
                label="Fuel Type"
                options={dictionary.fuelTypes}
                onChange={(key) => onFilterChange('fuelType', key)}
            />
        </div>
    );
}

function SelectFilter({ label, options, grouped, onChange }: {
    label: string;
    options: DictionaryEntry[];
    grouped?: boolean;
    onChange: (key: string) => void;
}) {
    const groups = grouped
        ? Object.groupBy(options, (o) => o.group ?? 'Other')
        : { [label]: options };

    return (
        <select onChange={(e) => onChange(e.target.value)}>
            <option value="">All {label}s</option>
            {Object.entries(groups).map(([group, items]) => (
                <optgroup key={group} label={group}>
                    {items!.map((item) => (
                        <option key={item.key} value={item.key}>
                            {item.label}
                        </option>
                    ))}
                </optgroup>
            ))}
        </select>
    );
}
```

### Sync Strategy

```
Backend (PHP)                    Frontend (TypeScript)
─────────────                    ─────────────────────
FilterDictionary DTO             FilterDictionary interface
  └─ DictionaryEntry               └─ DictionaryEntry
       ├─ key: string                    ├─ key: string
       ├─ label: string                  ├─ label: string
       └─ group: ?string                 └─ group?: string

       Source of truth ───────────► Derived from API response
       (Backend owns the schema)    (Frontend mirrors it)
```

> **Clean Code Tip:** Generate the TypeScript types from the backend's OpenAPI spec instead of maintaining them by hand. If the backend uses Swagger annotations (covered in [microservices-observability.md](./microservices-observability.md)), run `openapi-typescript` against the generated spec: `npx openapi-typescript ./openapi.json -o src/types/api.ts`. This eliminates the manual sync and catches schema drift at build time, not at runtime when a user sees a broken dropdown.
