---
layout: default
title: Adaptive Message Splitting for AWS SNS Size Limits
parent: Messaging & Event-Driven Architecture
nav_order: 6
---

## Adaptive Message Splitting for AWS SNS Size Limits

**Problem:** AWS SNS has a hard 256 KB per-message limit. When you publish a batch of items as a single message, some batches exceed this limit, but you don't know at serialization time which ones will. You also can't predict the encoded size before trying, because JSON encoding is data-dependent.

**Solution:** Use a generator to split the full payload into coarse initial chunks, then progressively split each chunk into smaller sub-parts until every part fits within the size limit. Combine this with a PHP generator for memory-efficient iteration over large payloads.

### The Splitter

Two responsibilities: initial chunking (memory-efficient, generator-based) and adaptive size splitting (retry loop):

```php
class BatchItemsSplitter
{
    private const ITEMS_PER_INITIAL_CHUNK = 500;

    /**
     * Splits a large item array into fixed-size chunks using a generator.
     * Memory usage stays flat - items are not all loaded at once.
     *
     * @param  Item[] $items
     * @return iterable<Item[]>
     */
    public function splitIntoChunks(array $items): iterable
    {
        $chunk = [];
        foreach ($items as $item) {
            if ($item->hasErrors()) {
                continue; // skip failed items
            }
            $chunk[] = $item;
            if (count($chunk) === self::ITEMS_PER_INITIAL_CHUNK) {
                yield $chunk;
                $chunk = [];
            }
        }
        if (!empty($chunk)) {
            yield $chunk;
        }
    }

    /**
     * Splits one chunk into $splitFactor equal parts.
     * splitFactor=1 returns the original chunk as a single-element array.
     * splitFactor=2 returns two halves, etc.
     *
     * @param  Item[] $chunk
     * @return Item[][]
     */
    public function splitChunkIntoParts(array $chunk, int $splitFactor = 1): array
    {
        if ($splitFactor === 1) {
            return [$chunk];
        }
        $partSize = (int) ceil(count($chunk) / $splitFactor);
        return array_chunk($chunk, $partSize);
    }
}
```

### The Producer: Adaptive Loop

For each initial chunk, try publishing it as a single message. If the serialized size exceeds the limit, increment `$splitFactor` and retry - splitting into more parts:

```php
class MessageProducer
{
    public function __construct(
        private readonly SnsClient        $sns,
        private readonly MessageSerializer $serializer,
        private readonly BatchItemsSplitter $splitter,
        private readonly string            $topicArn,
        private readonly int               $maxMessageBytes, // e.g. 256 * 1024
    ) {}

    public function publishBatch(array $items): void
    {
        foreach ($this->splitter->splitIntoChunks($items) as $chunk) {
            $this->publishChunkAdaptively($chunk);
        }
    }

    private function publishChunkAdaptively(array $chunk): void
    {
        $splitFactor = 0;

        while (true) {
            $splitFactor++;
            $parts       = $this->splitter->splitChunkIntoParts($chunk, $splitFactor);
            $oversized   = false;
            $messages    = [];

            foreach ($parts as $part) {
                $message = $this->buildMessage($part);
                $json    = $this->serializer->serialize($message);

                if (strlen($json) >= $this->maxMessageBytes) {
                    $oversized = true;
                    break; // this split factor is too large - try the next
                }
                $messages[] = [$message, $json];
            }

            if (!$oversized) {
                // All parts fit - publish them
                foreach ($messages as [$message]) {
                    $this->sns->publish($this->topicArn, $message);
                }
                return;
            }
        }
    }
}
```

### How the Adaptive Loop Works

```
Initial chunk: 500 items

splitFactor=1 → 1 part of 500 items → serialize → 300 KB → TOO BIG
splitFactor=2 → 2 parts of 250 items → serialize → 150 KB each → OK → publish 2 messages

(If both parts had been oversized, splitFactor=3 would try 167-item parts, and so on.)
```

For a chunk of `N` items:
- `splitFactor=1`: one message with all N items
- `splitFactor=2`: two messages with N/2 items each
- `splitFactor=k`: k messages with ceil(N/k) items each

The loop terminates because eventually `splitFactor` equals `N` (one item per message), which must fit in any reasonable size limit.

### Generator + Adaptive Split: Why Combine Them?

The generator (`splitIntoChunks`) solves the memory problem: instead of building one giant array of all items and then splitting it, items flow through one chunk at a time. The broader generator composition techniques behind this approach are covered in [Generator Patterns](../backend-patterns-optimization/generator-patterns.md).

The adaptive loop solves the size-estimation problem: you don't need to know item sizes in advance. Try, measure, adjust.

```
1,000 items
│
├── Generator yields chunk of 500
│     └── splitFactor=1 → too big
│     └── splitFactor=2 → fits → publish 2 SNS messages
│
└── Generator yields chunk of 500
      └── splitFactor=1 → fits → publish 1 SNS message
```

### Key Takeaways

| Design choice | Why |
|---|---|
| Generator for initial chunking | Flat memory regardless of total payload size |
| Adaptive `splitFactor` loop | No need to pre-compute message sizes; handles variable-size payloads |
| Re-serialize after each split | Guarantees actual byte count, not an estimate |
| Hard break on oversize | Avoids publishing a partial set of parts if any part is too big |

**Trade-off:** In the worst case (many very large items), the loop runs O(N) iterations where N is the chunk size. In practice this almost never happens - real-world items are small and `splitFactor=2` is usually enough. The simpler alternative - always splitting to fixed tiny chunks - wastes SNS API calls and adds latency for normal payloads.
