---
layout: default
title: ERP Bufferization
parent: Backend Patterns & Optimization for High-Volume Systems
nav_order: 5
---

## ERP Bufferization

**Problem:** A legacy ERP system (e.g., Oracle) has a hard limit on concurrent connections or API calls - say 50 requests/second. Your modern microservice processes 2,000 events/second. Sending each event individually will either crash the ERP or trigger rate-limit errors that cascade into retries and queue backlogs.

**Solution:** Buffer commands in memory (or a queue), flush them in controlled batches at a cadence the ERP can handle.

```php
class ErpCommandBuffer
{
    /** @var ErpCommand[] */
    private array $buffer = [];

    public function __construct(
        private ErpGateway $gateway,
        private int $batchSize = 50,
        private int $flushIntervalMs = 1000,
    ) {}

    public function add(ErpCommand $command): void
    {
        $this->buffer[] = $command;

        if (count($this->buffer) >= $this->batchSize) {
            $this->flush();
        }
    }

    public function flush(): void
    {
        if (empty($this->buffer)) {
            return;
        }

        $batch = array_splice($this->buffer, 0, $this->batchSize);

        try {
            $this->gateway->sendBatch($batch);
        } catch (ErpOverloadException $e) {
            // Re-queue failed batch for retry with backoff
            foreach ($batch as $command) {
                $this->buffer[] = $command;
            }

            usleep($this->flushIntervalMs * 1000);
            $this->flush();
        }
    }

    public function __destruct()
    {
        $this->flush(); // drain remaining on shutdown
    }
}
```

### Queue-Based Variant (Symfony Messenger)

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            erp_buffer:
                dsn: '%env(SQS_ERP_BUFFER_DSN)%'
                options:
                    wait_time: 10
                retry_strategy:
                    max_retries: 3
                    delay: 2000
                    multiplier: 3

        routing:
            App\Message\ErpCommand: erp_buffer
```

```php
// Consumer with rate limiting
#[AsMessageHandler]
class ErpCommandHandler
{
    public function __construct(
        private ErpGateway $gateway,
        private RateLimiterFactory $rateLimiterFactory,
    ) {}

    public function __invoke(ErpCommand $command): void
    {
        $limiter = $this->rateLimiterFactory->create('erp');

        // Block until a token is available (leaky bucket)
        $limiter->reserve(1)->wait();

        $this->gateway->send($command);
    }
}
```

> **Performance Tip:** The in-memory buffer is fast but loses data on process crash. For durability, use the queue-based approach (SQS/RabbitMQ) with a rate-limited consumer. The consumer pulls at the ERP's safe throughput (e.g., 50 msg/s), and the queue absorbs spikes. Monitor queue depth - if it grows steadily, your production rate permanently exceeds consumption rate and you need to negotiate higher ERP limits or aggregate commands.

