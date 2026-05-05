---
layout: default
title: Test Types
parent: Testing & Concurrency
nav_order: 1
---

## Test Types

Before writing a test, decide what kind of test you actually need. Mixing them up is the most common source of slow, fragile test suites.

| Type | What it tests | Mocks | Speed |
|---|---|---|---|
| **Unit** | A single function or class in isolation | Dependencies (via DI) | Very fast |
| **Integration** | Multiple units working together | External systems (DB, HTTP) | Medium |
| **Functional** | A full feature end-to-end | Nothing | Slow |
| **API** | HTTP contract of an endpoint | Downstream services | Medium |

---

### Unit Tests

Test a single class or function in complete isolation. All dependencies are replaced with mocks or stubs injected via the constructor. The test owns the mock - no global state, no real I/O.

The most common mistake: calling something a unit test while patching `global.fetch`, `axios`, or a database connection. Patching the transport layer is not isolation - the unit still depends on how that transport behaves internally. Inject the dependency instead, so the test controls exactly what the dependency returns.

**Scenario:** An `InvoiceCalculator` computes the invoice total from line items and a tax rate. The tax rate comes from a `TaxRateProvider` - an external config service. You want to test the calculation logic without involving the real config service.

```php
interface TaxRateProvider
{
    public function getRateForCountry(string $countryCode): float;
}

class InvoiceCalculator
{
    public function __construct(private TaxRateProvider $taxRates) {}

    public function calculate(array $lineItems, string $countryCode): InvoiceTotals
    {
        $subtotal = array_sum(array_column($lineItems, 'amount'));
        $taxRate  = $this->taxRates->getRateForCountry($countryCode);

        return new InvoiceTotals(
            subtotal: $subtotal,
            tax:      round($subtotal * $taxRate, 2),
            total:    round($subtotal * (1 + $taxRate), 2),
        );
    }
}
```

```php
class InvoiceCalculatorTest extends TestCase
{
    public function testCalculatesTotalWithTax(): void
    {
        $taxRates = $this->createMock(TaxRateProvider::class);
        $taxRates->method('getRateForCountry')
                 ->with('DE')
                 ->willReturn(0.19);

        $calculator = new InvoiceCalculator($taxRates);
        $totals = $calculator->calculate(
            lineItems: [['amount' => 100.00], ['amount' => 50.00]],
            countryCode: 'DE'
        );

        self::assertSame(150.00, $totals->subtotal);
        self::assertSame(28.50,  $totals->tax);
        self::assertSame(178.50, $totals->total);
    }

    public function testZeroTaxRateProducesNoTax(): void
    {
        $taxRates = $this->createMock(TaxRateProvider::class);
        $taxRates->method('getRateForCountry')->willReturn(0.0);

        $totals = (new InvoiceCalculator($taxRates))->calculate(
            lineItems: [['amount' => 200.00]],
            countryCode: 'AE'
        );

        self::assertSame(0.00,   $totals->tax);
        self::assertSame(200.00, $totals->total);
    }
}
```

The real `TaxRateProvider` (which might call an HTTP config service) never runs. The test fully controls what rate is returned, so failures point to the calculation logic, not the network.

See [JS Testing with Jest](jest-testing.md) for the TypeScript equivalent, and [Mocking with Prophecy](prophecy-promise-pattern.md) for Prophecy-based mocking.

---

### Integration Tests

Test multiple real components wired together against a real external system (database, message broker). The goal is to verify that the units collaborate correctly and that your assumptions about the external system hold.

What to mock: upstream HTTP services you don't control. What not to mock: your own database, your own queue.

**Scenario:** An `InvoiceRepository` saves and retrieves invoices. You want to verify the SQL, column mapping, and uniqueness constraint all work against a real database - not a mock that would silently pass even with a broken query.

```php
class InvoiceRepositoryIntegrationTest extends TestCase
{
    private Connection $db;
    private InvoiceRepository $repo;

    protected function setUp(): void
    {
        // Real test DB, rolled back after each test
        $this->db   = TestDatabase::connection();
        $this->repo = new InvoiceRepository($this->db);
        $this->db->beginTransaction();
    }

    protected function tearDown(): void
    {
        $this->db->rollBack();
    }

    public function testSaveAndRetrieveRoundtrip(): void
    {
        $invoice = new Invoice(
            number:      'INV-2024-001',
            customerId:  'cust-42',
            totalAmount: 178.50,
            currency:    'EUR',
        );

        $this->repo->save($invoice);
        $found = $this->repo->findByNumber('INV-2024-001');

        self::assertNotNull($found);
        self::assertSame('cust-42', $found->customerId);
        self::assertSame(178.50,   $found->totalAmount);
    }

    public function testDuplicateInvoiceNumberThrows(): void
    {
        $this->repo->save(new Invoice(number: 'INV-2024-002', customerId: 'c-1', totalAmount: 50.00, currency: 'EUR'));

        $this->expectException(UniqueConstraintViolationException::class);
        $this->repo->save(new Invoice(number: 'INV-2024-002', customerId: 'c-2', totalAmount: 99.00, currency: 'EUR'));
    }
}
```

Using `beginTransaction` + `rollBack` in `setUp`/`tearDown` keeps the database clean between tests without truncating tables. Each test runs in an isolated transaction that is never committed.

See [WireMock for Endpoint Testing](wiremock-endpoint-testing.md) for stubbing upstream HTTP calls when your integration test involves an outbound service call.

---

### Functional Tests

Test a complete feature from the outside - typically by driving the application through its public interface (HTTP, CLI) with a real database and real dependencies. Nothing is mocked. These tests are slow but give the highest confidence that the system actually works end-to-end.

**Scenario:** A payment is submitted. The full flow runs: HTTP request → controller → payment service → invoice created in DB → confirmation email queued. You assert both the HTTP response and the resulting database state.

```php
class SubmitPaymentFunctionalTest extends WebTestCase
{
    public function testPaymentCreatesInvoiceAndQueuesEmail(): void
    {
        $client = static::createClient();

        $client->request('POST', '/payments', [], [], [
            'CONTENT_TYPE' => 'application/json',
        ], json_encode([
            'customerId' => 'cust-99',
            'amount'     => 250.00,
            'currency'   => 'EUR',
            'method'     => 'card',
        ]));

        self::assertResponseStatusCodeSame(201);

        $body = json_decode($client->getResponse()->getContent(), true);
        self::assertArrayHasKey('invoiceNumber', $body);

        // Verify the invoice actually landed in the database
        $db      = static::getContainer()->get(Connection::class);
        $invoice = $db->fetchAssociative(
            'SELECT * FROM invoices WHERE number = ?',
            [$body['invoiceNumber']]
        );

        self::assertNotFalse($invoice);
        self::assertSame('cust-99', $invoice['customer_id']);
        self::assertSame('250.00',  $invoice['total_amount']);

        // Verify the confirmation email was queued
        $email = $db->fetchAssociative(
            'SELECT * FROM emails WHERE status = ? ORDER BY created_at DESC LIMIT 1',
            [EmailStatus::QUEUE->value]
        );

        self::assertNotFalse($email);
        self::assertStringContainsString($body['invoiceNumber'], $email['subject']);
    }
}
```

This test exercises every layer. If the controller misroutes the request, the service has a bug, the DB schema is wrong, or the email is never queued - the test fails. The trade-off is speed: it requires a running application and a real database, so it belongs in a separate suite run less frequently than unit tests.

---

### API Tests

Verify the HTTP contract of your own endpoints: correct status codes, response shapes, headers, and error responses. Run against a live application instance (local or staging). Downstream services are stubbed (WireMock, recorded cassettes) so the tests are deterministic.

The distinction from functional tests: API tests focus on the contract your service exposes, not on internal behaviour. A functional test might also assert DB state or side effects; an API test stops at the HTTP boundary.

**Scenario:** `GET /api/v1/invoices/{id}` fetches an invoice and enriches it with the customer's current credit status from an external CRM. You want to verify the response shape and that a CRM failure degrades gracefully - without depending on the real CRM.

```php
class InvoiceApiTest extends TestCase
{
    public function testReturnsInvoiceWithCreditStatus(): void
    {
        // WireMock stub - loaded from tests/wiremock/mappings/
        // GET /crm/customers/cust-42/credit → 200 {"status": "good"}

        $response = $this->apiClient->get('/api/v1/invoices/INV-2024-001');

        self::assertSame(200, $response->getStatusCode());

        $body = $response->toArray();
        self::assertSame('INV-2024-001', $body['number']);
        self::assertSame(178.50,         $body['totalAmount']);
        self::assertSame('good',         $body['customerCreditStatus']);
    }

    public function testReturns404ForUnknownInvoice(): void
    {
        $response = $this->apiClient->get('/api/v1/invoices/INV-DOES-NOT-EXIST');

        self::assertSame(404, $response->getStatusCode());
        self::assertSame('invoice_not_found', $response->toArray()['error']);
    }

    public function testDegradesgracefullyWhenCrmIsDown(): void
    {
        // Override WireMock stub for this test only
        $this->registerWireMockStub([
            'request'  => ['method' => 'GET', 'urlPath' => '/crm/customers/cust-42/credit'],
            'response' => ['status' => 503],
        ]);

        $response = $this->apiClient->get('/api/v1/invoices/INV-2024-001');

        // Invoice still returned, credit status falls back to null
        self::assertSame(200,  $response->getStatusCode());
        self::assertNull($response->toArray()['customerCreditStatus']);
    }
}
```

The CRM is never called - WireMock intercepts it. The test is deterministic regardless of CRM uptime, and it documents the expected degradation behaviour as executable specification.

---

### For AI agents

```
Four types: Unit (single class, mock injected deps via DI, no I/O), Integration (real DB/queue, stub external HTTP with WireMock, transaction rollback for cleanup), Functional (full HTTP flow, real DB, nothing mocked, assert DB state), API (HTTP contract, stub downstream services, verify status codes and response shapes). Never call a test "unit" if it touches the network, filesystem, or database.
```

Reference: `https://michalsniezko.github.io/testing-concurrency-locks/test-types.html`
