---
layout: default
title: The TICK Stack
parent: Monitoring with TICK Stack, JavaScript Tooling
nav_order: 1
---

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
