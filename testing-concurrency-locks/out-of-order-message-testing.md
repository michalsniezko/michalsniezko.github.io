---
layout: default
title: Out-of-Order Message Testing
parent: Testing & Concurrency
nav_order: 6
---

## Out-of-Order Message Testing

**Scenario:** SQS delivers two messages for shipment `s-123`: first `status=delivered` (timestamp `T2`), then `status=dispatched` (timestamp `T1`, older). The timestamp-based idempotency approach being tested here is described in [Handling Out-of-Order Messages](../aws-event-driven/out-of-order.md). If your consumer blindly applies each message, the shipment ends up as "dispatched" when it should be "delivered." Your test must prove the consumer correctly discards stale events.

### Consumer Logic (Recap)

```php
class ShipmentStatusConsumer
{
    public function __construct(private Connection $db) {}

    public function handle(array $message): void
    {
        $this->db->executeStatement(
            'INSERT INTO shipment_status (external_id, status, event_timestamp)
             VALUES (:id, :status, :ts)
             ON CONFLICT (external_id)
             DO UPDATE SET
                 status          = EXCLUDED.status,
                 event_timestamp = EXCLUDED.event_timestamp
             WHERE shipment_status.event_timestamp < EXCLUDED.event_timestamp',
            [
                'id'     => $message['entity_id'],
                'status' => $message['payload']['status'],
                'ts'     => $message['event_timestamp'],
            ]
        );
    }
}
```

### PHPUnit Test: Out-of-Order Delivery

```php
class ShipmentStatusConsumerTest extends TestCase
{
    private Connection $db;
    private ShipmentStatusConsumer $consumer;

    protected function setUp(): void
    {
        $this->db = TestDatabaseFactory::create(); // test DB with migrations applied
        $this->consumer = new ShipmentStatusConsumer($this->db);
    }

    public function testNewerEventWins(): void
    {
        // Deliver T2 first (newer event arrives first)
        $this->consumer->handle([
            'entity_id'       => 's-123',
            'event_timestamp' => '2026-03-06T14:00:00Z',
            'payload'         => ['status' => 'delivered'],
        ]);

        // Deliver T1 second (older event arrives late)
        $this->consumer->handle([
            'entity_id'       => 's-123',
            'event_timestamp' => '2026-03-06T13:00:00Z',
            'payload'         => ['status' => 'dispatched'],
        ]);

        // Assert: the newer status survives
        $row = $this->db->fetchAssociative(
            'SELECT status, event_timestamp FROM shipment_status WHERE external_id = :id',
            ['id' => 's-123']
        );

        self::assertSame('delivered', $row['status']);
        self::assertSame('2026-03-06T14:00:00Z', $row['event_timestamp']);
    }

    public function testOlderEventIsAppliedWhenNoRowExists(): void
    {
        // First message for this entity - should always insert regardless of age
        $this->consumer->handle([
            'entity_id'       => 's-456',
            'event_timestamp' => '2026-03-06T10:00:00Z',
            'payload'         => ['status' => 'dispatched'],
        ]);

        $row = $this->db->fetchAssociative(
            'SELECT status FROM shipment_status WHERE external_id = :id',
            ['id' => 's-456']
        );

        self::assertSame('dispatched', $row['status']);
    }

    public function testSameTimestampDoesNotOverwrite(): void
    {
        $ts = '2026-03-06T14:00:00Z';

        $this->consumer->handle([
            'entity_id'       => 's-789',
            'event_timestamp' => $ts,
            'payload'         => ['status' => 'delivered'],
        ]);

        // Same timestamp, different status - should NOT overwrite (< not <=)
        $this->consumer->handle([
            'entity_id'       => 's-789',
            'event_timestamp' => $ts,
            'payload'         => ['status' => 'ready_for_pickup'],
        ]);

        $row = $this->db->fetchAssociative(
            'SELECT status FROM shipment_status WHERE external_id = :id',
            ['id' => 's-789']
        );

        self::assertSame('delivered', $row['status']);
    }

    /**
     * Simulate realistic SQS delivery: randomize the order of N events
     * and assert the final state always reflects the newest event.
     */
    public function testRandomizedDeliveryOrder(): void
    {
        $events = [
            ['ts' => '2026-03-06T10:00:00Z', 'status' => 'dispatched'],
            ['ts' => '2026-03-06T11:00:00Z', 'status' => 'in_transit'],
            ['ts' => '2026-03-06T12:00:00Z', 'status' => 'out_for_delivery'],
            ['ts' => '2026-03-06T13:00:00Z', 'status' => 'attempted_delivery'],
            ['ts' => '2026-03-06T14:00:00Z', 'status' => 'delivered'],
        ];

        // Run 10 times with different shuffles
        for ($i = 0; $i < 10; $i++) {
            $this->db->executeStatement("DELETE FROM shipment_status WHERE external_id = 's-shuffle'");
            $shuffled = $events;
            shuffle($shuffled);

            foreach ($shuffled as $event) {
                $this->consumer->handle([
                    'entity_id'       => 's-shuffle',
                    'event_timestamp' => $event['ts'],
                    'payload'         => ['status' => $event['status']],
                ]);
            }

            $row = $this->db->fetchAssociative(
                'SELECT status, event_timestamp FROM shipment_status WHERE external_id = :id',
                ['id' => 's-shuffle']
            );

            self::assertSame('delivered', $row['status'], "Failed on shuffle iteration $i");
            self::assertSame('2026-03-06T14:00:00Z', $row['event_timestamp']);
        }
    }
}
```

> **Safety First:** The randomized test (`testRandomizedDeliveryOrder`) is a poor man's property-based test. It catches ordering bugs, but `shuffle()` is non-deterministic - a failing run might not reproduce. Log the shuffle order on failure so you can replay the exact sequence. For deterministic reproduction, seed the shuffle: `mt_srand($seed); shuffle($shuffled);` and log the seed.

---

### For AI agents

```
When testing SQS consumers that handle out-of-order delivery: write tests that send messages in reverse or random order and assert only the latest state is applied. Verify older messages are discarded, not applied on top of newer ones.
```

Reference: `https://michalsniezko.github.io/testing-concurrency-locks/out-of-order-message-testing.html`
