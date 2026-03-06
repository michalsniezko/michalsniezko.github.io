---
layout: default
title: Backend Patterns
nav_order: 4
---

# Backend Patterns & Optimization for High-Volume Systems

## Generator Patterns with `yield from`

**Problem:** Loading 500k rows into an array to transform and filter them will eat gigabytes of memory. `array_map` and `array_filter` both build full intermediate arrays in memory - you pay for the entire dataset even if you only need the first 100 matching rows.

**Solution:** Generators produce one item at a time. Memory usage stays flat regardless of dataset size. `yield from` lets you compose generators by delegating to sub-generators without flattening into arrays.

```php
function readCsvRows(string $filePath): Generator
{
    $handle = fopen($filePath, 'r');
    fgetcsv($handle); // skip header

    while (($row = fgetcsv($handle)) !== false) {
        yield [
            'email'  => $row[0],
            'name'   => $row[1],
            'amount' => (float) $row[2],
        ];
    }

    fclose($handle);
}

function filterHighValue(Generator $rows, float $threshold): Generator
{
    foreach ($rows as $row) {
        if ($row['amount'] >= $threshold) {
            yield $row;
        }
    }
}

function mergeDataSources(array $filePaths): Generator
{
    foreach ($filePaths as $path) {
        yield from readCsvRows($path);
    }
}

// Process 3 files with 500k rows each - memory stays under 2MB
$allRows = mergeDataSources([
    '/data/transactions_jan.csv',
    '/data/transactions_feb.csv',
    '/data/transactions_mar.csv',
]);

$highValue = filterHighValue($allRows, 1000.00);

foreach ($highValue as $row) {
    $exporter->push($row); // stream out one row at a time
}
```

### Why `yield from` Instead of Nested Loops

```php
// Without yield from - you must manually iterate and re-yield
function merged(array $sources): Generator
{
    foreach ($sources as $source) {
        foreach (readCsvRows($source) as $row) {
            yield $row; // boilerplate
        }
    }
}

// With yield from - delegation is clean
function merged(array $sources): Generator
{
    foreach ($sources as $source) {
        yield from readCsvRows($source);
    }
}
```

`yield from` also propagates the return value of the sub-generator (via `Generator::getReturn()`), which is useful for collecting summaries like row counts.

> **Performance Tip:** A generator processing 1M rows uses ~2MB of memory. The same pipeline with `array_map` + `array_filter` allocates ~200MB+ for intermediate arrays. The trade-off: you lose random access (`$rows[500]`) and can only iterate once. If you need multiple passes, either re-create the generator or `iterator_to_array()` a small, filtered subset.

<pre class="mermaid">
    %%{init: {'theme':'neutral'}}%%
sequenceDiagram
    participant App as Main Loop (foreach)
    participant Merger as mergeDataSources()<br/>(The Orchestrator)
    participant Reader as readCsvRows()<br/>(The Producer)
    participant Filter as filterHighValue()<br/>(The Processor)
    participant File as CSV File on Disk

    Note over App,File: START: Memory usage ~2MB
    
    App->>Merger: Request next row (foreach)
    activate Merger
    Note right of Merger: yield from delegates to<br/>readCsvRows('jan.csv')
    Merger->>Reader: Request first row
    activate Reader
    
    Reader->>File: Read one line
    File-->>Reader: Line: [email, name, 1500.00]
    Note right of Reader: Yields: ['amount' => 1500.00]
    Reader-->>Merger: (Delegated via yield from)
    deactivate Reader
    
    Merger-->>Filter: Passes row to filter
    activate Filter
    Note right of Filter: Checks: 1500.00 >= 1000.00?<br/>YES
    Note right of Filter: Yields: ['amount' => 1500.00]
    Filter-->>App: (Passes to final foreach)
    deactivate Filter
    
    App->>Exporter: push($row)
    Note over App,File: Row Processed. Memory usage still ~2MB

    %% -- Row 2 (Filtered Out) --
    
    App->>Merger: Request next row
    activate Merger
    Merger->>Reader: Request next row
    activate Reader
    Reader->>File: Read one line
    File-->>Reader: Line: [email, name, 500.00]
    Note right of Reader: Yields: ['amount' => 500.00]
    Reader-->>Merger: (Delegated)
    deactivate Reader
    
    Merger-->>Filter: Passes row to filter
    activate Filter
    Note right of Filter: Checks: 500.00 >= 1000.00?<br/>NO. No yield happens.
    Note over Filter: Filter continues its internal foreach<br/>to find next match.
    deactivate Filter

    Note over App,File: The pipeline "pauses" until Filter yields<br/>or Reader reaches EOF.
</pre>
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

---

## Prophecy & Promise Pattern (PHPUnit)

**Problem:** Traditional mocking with `$this->createMock()` ties tests to method call counts and argument order. Tests break when internal implementation changes even though behavior is correct. The result: brittle tests that punish refactoring.

**Solution:** Prophecy focuses on behavioral specification. You describe what the collaborator *should do* when called, not the exact mechanics. Promises (`willReturn`, `willThrow`) define behavior; predictions (`shouldBeCalled`) verify interactions only when necessary.

```php
use PHPUnit\Framework\TestCase;
use Prophecy\PhpUnit\ProphecyTrait;

class OrderServiceTest extends TestCase
{
    use ProphecyTrait;

    public function testCreateOrderCalculatesTotal(): void
    {
        // Arrange: define collaborator behavior
        $pricingClient = $this->prophesize(PricingClient::class);
        $pricingClient
            ->calculatePrice('product-abc', 3)
            ->willReturn(new PriceDTO(unitPrice: 29.99, total: 89.97));

        $orderRepository = $this->prophesize(OrderRepository::class);
        $orderRepository
            ->save(Argument::type(Order::class))
            ->shouldBeCalledOnce();

        // Act
        $service = new OrderService(
            $pricingClient->reveal(),
            $orderRepository->reveal(),
        );

        $order = $service->createOrder('product-abc', quantity: 3);

        // Assert
        self::assertSame(89.97, $order->getTotal());
    }

    public function testCreateOrderHandlesPricingFailure(): void
    {
        $pricingClient = $this->prophesize(PricingClient::class);
        $pricingClient
            ->calculatePrice(Argument::cetera())
            ->willThrow(new ServiceUnavailableException('Pricing service down'));

        $orderRepository = $this->prophesize(OrderRepository::class);
        $orderRepository->save(Argument::any())->shouldNotBeCalled();

        $service = new OrderService(
            $pricingClient->reveal(),
            $orderRepository->reveal(),
        );

        $this->expectException(OrderCreationException::class);
        $service->createOrder('product-abc', quantity: 3);
    }
}
```

### Key Prophecy Methods

| Method                        | Purpose                                        |
|-------------------------------|------------------------------------------------|
| `willReturn($value)`         | Return a value when called                     |
| `willThrow($exception)`      | Throw when called                              |
| `shouldBeCalled()`           | Fail if never called (prediction)              |
| `shouldBeCalledOnce()`       | Fail if called zero or 2+ times               |
| `shouldNotBeCalled()`        | Fail if called at all                          |
| `Argument::type(Foo::class)` | Match any argument of type `Foo`               |
| `Argument::cetera()`         | Match any remaining arguments                  |

> **Performance Tip:** Prophecy is deprecated in PHPUnit 10+. If you're on PHPUnit 10, either require `phpspec/prophecy-phpunit` as a bridge or migrate to PHPUnit's native `createStub()` / `createMock()` - which have improved significantly. For new projects, prefer `createStub()` for behavior (no call verification) and `createMock()` only when you need `expects($this->once())`.

---

## WireMock for Endpoint Testing

**Problem:** Integration tests that hit real external services are slow, flaky, and dependent on third-party uptime. Mocking at the PHP level (Prophecy/PHPUnit) skips the HTTP layer entirely - you miss serialization bugs, wrong headers, and real HTTP status handling.

**Solution:** WireMock runs a real HTTP server that returns canned responses. Your application makes actual HTTP calls to `localhost:8080` (WireMock), so the full HTTP client stack is exercised.

### WireMock Stub Definition

```json
{
  "request": {
    "method": "GET",
    "urlPathPattern": "/api/v1/vehicles/[a-f0-9-]+",
    "headers": {
      "Accept": { "equalTo": "application/json" }
    }
  },
  "response": {
    "status": 200,
    "headers": { "Content-Type": "application/json" },
    "jsonBody": {
      "id": "abc-123",
      "make": "Toyota",
      "model": "Corolla",
      "year": 2024
    }
  }
}
```

### Docker Compose Setup

```yaml
# docker-compose.test.yml
services:
  wiremock:
    image: wiremock/wiremock:3.5.4
    ports:
      - "8080:8080"
    volumes:
      - ./tests/wiremock/mappings:/home/wiremock/mappings
      - ./tests/wiremock/__files:/home/wiremock/__files
    command: --verbose
```

### Test Using WireMock

```php
class VehicleClientIntegrationTest extends TestCase
{
    private VehicleClient $client;

    protected function setUp(): void
    {
        $this->client = new VehicleClient(
            HttpClient::create(),
            'http://localhost:8080' // WireMock
        );
    }

    public function testGetVehicleReturnsDTO(): void
    {
        // WireMock mapping already loaded from ./tests/wiremock/mappings/
        $vehicle = $this->client->getVehicle('abc-123');

        self::assertSame('Toyota', $vehicle->make);
        self::assertSame(2024, $vehicle->year);
    }

    public function testGetVehicleHandles500(): void
    {
        // Use WireMock admin API to register fault scenario
        $this->registerWireMockStub([
            'request' => ['method' => 'GET', 'urlPath' => '/api/v1/vehicles/fail-id'],
            'response' => ['status' => 500, 'body' => 'Internal Server Error'],
        ]);

        $this->expectException(UpstreamServiceException::class);
        $this->client->getVehicle('fail-id');
    }

    private function registerWireMockStub(array $mapping): void
    {
        HttpClient::create()->request('POST', 'http://localhost:8080/__admin/mappings', [
            'json' => $mapping,
        ]);
    }
}
```

> **Gotcha:** WireMock stubs are stateful per server instance. If tests run in parallel, one test's stub can leak into another. Use `"scenarioName"` for stateful sequences, and call `POST /__admin/reset` in `setUp()` if tests share a WireMock instance. Or use `docker compose` to spin up an isolated instance per test suite.

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

---

## Nested Dictionary DTOs

**Problem:** The frontend renders dynamic filter dropdowns (vehicle make, color, fuel type) that need a structured list of possible values. Hardcoding these on the frontend means deploying a JS build every time a value changes. Sending raw DB enums is messy and leaks internal naming.

**Solution:** Return a single "dictionary" endpoint that nests all filterable dimensions into a typed DTO. The frontend calls it once on page load and populates all dropdowns from the response.

### DTO Structure

```php
class FilterDictionary
{
    public function __construct(
        /** @var DictionaryEntry[] */
        public readonly array $makes,
        /** @var DictionaryEntry[] */
        public readonly array $fuelTypes,
        /** @var DictionaryEntry[] */
        public readonly array $colors,
        /** @var DictionaryEntry[] */
        public readonly array $transmissions,
    ) {}
}

class DictionaryEntry
{
    public function __construct(
        public readonly string $key,   // internal value: "diesel"
        public readonly string $label, // display value: "Diesel"
        public readonly ?string $group = null, // optional grouping: "German" for makes
    ) {}
}
```

### Builder

```php
class FilterDictionaryBuilder
{
    public function build(string $locale): FilterDictionary
    {
        return new FilterDictionary(
            makes: $this->buildMakes(),
            fuelTypes: [
                new DictionaryEntry('petrol', 'Petrol'),
                new DictionaryEntry('diesel', 'Diesel'),
                new DictionaryEntry('electric', 'Electric'),
                new DictionaryEntry('hybrid', 'Hybrid'),
            ],
            colors: $this->buildColors($locale),
            transmissions: [
                new DictionaryEntry('manual', 'Manual'),
                new DictionaryEntry('automatic', 'Automatic'),
            ],
        );
    }

    private function buildMakes(): array
    {
        return [
            new DictionaryEntry('bmw', 'BMW', group: 'German'),
            new DictionaryEntry('mercedes', 'Mercedes-Benz', group: 'German'),
            new DictionaryEntry('toyota', 'Toyota', group: 'Japanese'),
            new DictionaryEntry('honda', 'Honda', group: 'Japanese'),
        ];
    }

    private function buildColors(string $locale): array
    {
        // In production, load translations from a catalog
        return [
            new DictionaryEntry('black', $locale === 'de' ? 'Schwarz' : 'Black'),
            new DictionaryEntry('white', $locale === 'de' ? 'Weiß' : 'White'),
            new DictionaryEntry('silver', $locale === 'de' ? 'Silber' : 'Silver'),
        ];
    }
}
```

### API Response

```json
{
  "makes": [
    {"key": "bmw", "label": "BMW", "group": "German"},
    {"key": "mercedes", "label": "Mercedes-Benz", "group": "German"},
    {"key": "toyota", "label": "Toyota", "group": "Japanese"}
  ],
  "fuelTypes": [
    {"key": "petrol", "label": "Petrol"},
    {"key": "diesel", "label": "Diesel"},
    {"key": "electric", "label": "Electric"}
  ],
  "colors": [
    {"key": "black", "label": "Black"},
    {"key": "white", "label": "White"}
  ],
  "transmissions": [
    {"key": "manual", "label": "Manual"},
    {"key": "automatic", "label": "Automatic"}
  ]
}
```

### Controller

```php
#[Route('/api/v1/filters/dictionaries', methods: ['GET'])]
public function getDictionaries(
    Request $request,
    FilterDictionaryBuilder $builder,
): JsonResponse {
    $locale = $request->query->get('locale', 'en');
    $dictionary = $builder->build($locale);

    return $this->json($dictionary, 200, [
        'Cache-Control' => 'public, max-age=300', // 5 min cache
    ]);
}
```

> **Performance Tip:** Dictionary data changes rarely. Set `Cache-Control` headers aggressively (5–15 min) and consider a CDN or reverse proxy (Varnish) in front of this endpoint. For large dictionaries (10k+ vehicle models), add an `ETag` header based on a hash of the data so clients skip re-downloading unchanged payloads (`304 Not Modified`).

<script type="module">
	import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
	mermaid.initialize({
		startOnLoad: true,
		theme: 'default'
	});
</script>
