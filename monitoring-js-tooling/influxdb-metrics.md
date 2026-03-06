---
layout: default
title: InfluxDB for Metrics
parent: Monitoring with TICK Stack, JavaScript Tooling
nav_order: 2
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
