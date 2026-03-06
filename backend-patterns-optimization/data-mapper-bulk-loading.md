---
layout: default
title: Data Mapper with Bulk Loading
parent: Backend Patterns & Optimization for High-Volume Systems
nav_order: 2
---

## Data Mapper with Bulk Loading

**Problem:** You have 200 orders. Each order has a `merchantId`. Fetching each merchant one-by-one from the Merchant Service means 200 HTTP calls (N+1 problem). At 50ms per call, that's 10 seconds of latency just for hydration.

**Solution:** Collect all unique IDs, make one bulk API call, index the result by ID, then map back. This is the "collect → fetch → hydrate" pattern.

```php
class OrderHydrator
{
    public function __construct(
        private MerchantClient $merchantClient,
    ) {}

    /**
     * @param OrderDTO[] $orders
     * @return HydratedOrderDTO[]
     */
    public function hydrateWithMerchants(array $orders): array
    {
        // 1. Collect unique IDs
        $merchantIds = array_unique(
            array_map(fn(OrderDTO $o) => $o->merchantId, $orders)
        );

        if (empty($merchantIds)) {
            return $this->mapWithoutMerchants($orders);
        }

        // 2. Bulk fetch (single HTTP call)
        $merchants = $this->merchantClient->getByIds($merchantIds);

        // 3. Index by ID for O(1) lookup
        $merchantMap = [];
        foreach ($merchants as $merchant) {
            $merchantMap[$merchant->id] = $merchant;
        }

        // 4. Hydrate
        return array_map(
            fn(OrderDTO $order) => new HydratedOrderDTO(
                order: $order,
                merchant: $merchantMap[$order->merchantId] ?? null,
            ),
            $orders
        );
    }
}
```

### The Bulk Client

```php
class MerchantClient
{
    public function __construct(
        private HttpClientInterface $httpClient,
        private string $baseUrl,
    ) {}

    /** @return MerchantDTO[] */
    public function getByIds(array $ids): array
    {
        $response = $this->httpClient->request('POST', $this->baseUrl . '/api/v1/merchants/bulk', [
            'json' => ['ids' => array_values($ids)],
        ]);

        return array_map(
            fn(array $data) => MerchantDTO::fromArray($data),
            $response->toArray()
        );
    }
}
```

> **Performance Tip:** Most bulk APIs have a max batch size (commonly 100–500 IDs). If you have 2,000 unique IDs, chunk them: `array_chunk($ids, 200)` and fire requests in parallel using Symfony HttpClient's async capabilities. 10 parallel requests of 200 IDs each finishes in ~50ms instead of 10 sequential requests at ~500ms.

