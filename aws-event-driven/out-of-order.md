---
layout: default
title: Handling Out-of-Order Messages
parent: AWS Event-Driven Architecture
nav_order: 3
---


## Handling Out-of-Order Messages

**Problem:** Standard SQS queues do not guarantee message ordering. If you publish `order.updated (status=paid)` and then `order.updated (status=shipped)` in quick succession, the consumer might process `shipped` before `paid`. Your database now says "paid" when it should say "shipped."

**Approach: Timestamp-based idempotency at the consumer.**

Every message includes an `event_timestamp`. The consumer compares the incoming timestamp against the last-processed timestamp stored alongside the entity. If the incoming event is older, it's discarded.

### Message Schema

```json
{
  "event_type": "order.updated",
  "entity_id": "order-abc-123",
  "event_timestamp": "2026-03-06T14:30:00.123Z",
  "payload": {
    "status": "shipped",
    "tracking_number": "DHL-987654"
  }
}
```

### Consumer Pseudocode

```python
def handle_order_update(message):
    order = db.get_order(message["entity_id"])

    if order and order.last_event_at >= message["event_timestamp"]:
        log.info(f"Skipping stale event for {message['entity_id']}")
        return  # ACK the message, discard stale data

    db.upsert_order(
        entity_id=message["entity_id"],
        status=message["payload"]["status"],
        last_event_at=message["event_timestamp"]
    )
```

> **Gotcha:** Use the publisher's event timestamp, **not** the SQS `SentTimestamp` attribute. SQS timestamps reflect when the message entered the queue, not when the business event occurred. If a publisher retries a failed SNS publish, the SQS timestamp will be newer but the event is the same (or older).

> **Pro-Tip:** For strict ordering without consumer-side logic, use **SQS FIFO queues** with `MessageGroupId`. Messages within the same group are delivered in order. Trade-off: FIFO queues cap at 300 msg/s per group (3,000 with batching) vs. virtually unlimited for standard queues.
