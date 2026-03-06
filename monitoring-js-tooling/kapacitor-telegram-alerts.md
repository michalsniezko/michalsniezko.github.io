---
layout: default
title: Kapacitor & Telegram Alerts
parent: Monitoring with TICK Stack, JavaScript Tooling
nav_order: 3
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
