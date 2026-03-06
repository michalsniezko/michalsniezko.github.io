---
layout: default
title: Out-of-Order Message Testing
parent: Testing, Concurrency, Distributed Locks
nav_order: 6
---

## Out-of-Order Message Testing

**Scenario:** SQS delivers two messages for vehicle `v-123`: first `status=inspected` (timestamp `T2`), then `status=received` (timestamp `T1`, older). If your consumer blindly applies each message, the vehicle ends up as "received" when it should be "inspected." Your test must prove the consumer correctly discards stale events.

### Consumer Logic (Recap)

```php
class VehicleStatusConsumer
{
    public function __construct(private Connection $db) {}

    public function handle(array $message): void
    {
        $this->db->executeStatement(
            'INSERT INTO vehicle_status (external_id, status, event_timestamp)
             VALUES (:id, :status, :ts)
             ON CONFLICT (external_id)
             DO UPDATE SET
                 status          = EXCLUDED.status,
                 event_timestamp = EXCLUDED.event_timestamp
             WHERE vehicle_status.event_timestamp < EXCLUDED.event_timestamp',
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
class VehicleStatusConsumerTest extends TestCase
{
    private Connection $db;
    private VehicleStatusConsumer $consumer;

    protected function setUp(): void
    {
        $this->db = TestDatabaseFactory::create(); // test DB with migrations applied
        $this->consumer = new VehicleStatusConsumer($this->db);
    }

    public function testNewerEventWins(): void
    {
        // Deliver T2 first (newer event arrives first)
        $this->consumer->handle([
            'entity_id'       => 'v-123',
            'event_timestamp' => '2026-03-06T14:00:00Z',
            'payload'         => ['status' => 'inspected'],
        ]);

        // Deliver T1 second (older event arrives late)
        $this->consumer->handle([
            'entity_id'       => 'v-123',
            'event_timestamp' => '2026-03-06T13:00:00Z',
            'payload'         => ['status' => 'received'],
        ]);

        // Assert: the newer status survives
        $row = $this->db->fetchAssociative(
            'SELECT status, event_timestamp FROM vehicle_status WHERE external_id = :id',
            ['id' => 'v-123']
        );

        self::assertSame('inspected', $row['status']);
        self::assertSame('2026-03-06T14:00:00Z', $row['event_timestamp']);
    }

    public function testOlderEventIsAppliedWhenNoRowExists(): void
    {
        // First message for this entity - should always insert regardless of age
        $this->consumer->handle([
            'entity_id'       => 'v-456',
            'event_timestamp' => '2026-03-06T10:00:00Z',
            'payload'         => ['status' => 'received'],
        ]);

        $row = $this->db->fetchAssociative(
            'SELECT status FROM vehicle_status WHERE external_id = :id',
            ['id' => 'v-456']
        );

        self::assertSame('received', $row['status']);
    }

    public function testSameTimestampDoesNotOverwrite(): void
    {
        $ts = '2026-03-06T14:00:00Z';

        $this->consumer->handle([
            'entity_id'       => 'v-789',
            'event_timestamp' => $ts,
            'payload'         => ['status' => 'inspected'],
        ]);

        // Same timestamp, different status - should NOT overwrite (< not <=)
        $this->consumer->handle([
            'entity_id'       => 'v-789',
            'event_timestamp' => $ts,
            'payload'         => ['status' => 'ready_for_sale'],
        ]);

        $row = $this->db->fetchAssociative(
            'SELECT status FROM vehicle_status WHERE external_id = :id',
            ['id' => 'v-789']
        );

        self::assertSame('inspected', $row['status']);
    }

    /**
     * Simulate realistic SQS delivery: randomize the order of N events
     * and assert the final state always reflects the newest event.
     */
    public function testRandomizedDeliveryOrder(): void
    {
        $events = [
            ['ts' => '2026-03-06T10:00:00Z', 'status' => 'received'],
            ['ts' => '2026-03-06T11:00:00Z', 'status' => 'inspected'],
            ['ts' => '2026-03-06T12:00:00Z', 'status' => 'ready_for_sale'],
            ['ts' => '2026-03-06T13:00:00Z', 'status' => 'reserved'],
            ['ts' => '2026-03-06T14:00:00Z', 'status' => 'sold'],
        ];

        // Run 10 times with different shuffles
        for ($i = 0; $i < 10; $i++) {
            $this->db->executeStatement("DELETE FROM vehicle_status WHERE external_id = 'v-shuffle'");
            $shuffled = $events;
            shuffle($shuffled);

            foreach ($shuffled as $event) {
                $this->consumer->handle([
                    'entity_id'       => 'v-shuffle',
                    'event_timestamp' => $event['ts'],
                    'payload'         => ['status' => $event['status']],
                ]);
            }

            $row = $this->db->fetchAssociative(
                'SELECT status, event_timestamp FROM vehicle_status WHERE external_id = :id',
                ['id' => 'v-shuffle']
            );

            self::assertSame('sold', $row['status'], "Failed on shuffle iteration $i");
            self::assertSame('2026-03-06T14:00:00Z', $row['event_timestamp']);
        }
    }
}
```

> **Safety First:** The randomized test (`testRandomizedDeliveryOrder`) is a poor man's property-based test. It catches ordering bugs, but `shuffle()` is non-deterministic - a failing run might not reproduce. Log the shuffle order on failure so you can replay the exact sequence. For deterministic reproduction, seed the shuffle: `mt_srand($seed); shuffle($shuffled);` and log the seed.
