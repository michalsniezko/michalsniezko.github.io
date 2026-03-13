---
layout: default
title: Specification Pattern for Complex Financial Validation
parent: PHP Patterns in Practice
nav_order: 7
---

## Specification Pattern for Complex Financial Validation

**Problem:** Financial business rules are complex, conditional, and change frequently. Putting all the `if` statements inside a service method creates an unreadable wall of code. Adding a new rule requires editing a method that already has 10 conditions. Testing one rule in isolation is impossible because they're all entangled.

**Solution:** Extract each business rule into its own Specification class with a single `isSatisfied()` method. Compose specifications in a higher-level service. Use a Guard for pre-flight checks that should throw rather than return false.

### A Specification

Each specification answers one yes/no question about a domain object:

```php
final class OrderPaymentReadySpecification
{
    public function __construct(
        private readonly ProductDetailsRepository  $productDetails,
        private readonly OrderPurchaseRepository   $orderPurchase,
        private readonly PaymentBlockerRepository  $blockers,
    ) {}

    public function isSatisfied(Order $order): bool
    {
        $productId = $order->getProductId();

        // Rule 1: Discounted orders only check for payment blockers
        if ($this->isDiscountedOrder($order)) {
            return !$this->hasDisallowedBlockers($productId);
        }

        // Rule 2: Non-digital-product orders are always ready
        if (!$order->isDigitalProduct()) {
            return true;
        }

        // Rule 3: For digital-product orders, apply the full rule set
        $isRegular  = $this->isNotExemptFromFees($productId);
        $hasBlockers = $this->hasDisallowedBlockers($productId);

        if ($isRegular && $hasBlockers) {
            return false;
        }

        return $this->isGuestCheckout($order->getCustomerId()) || !$hasBlockers;
    }

    private function isDiscountedOrder(Order $order): bool
    {
        return $order->isDiscountedPricing()
            && $order->isExemptFromFees()
            && $this->isHighValueOrder($order->getProductId())
            && !$this->isGuestCheckout($order->getCustomerId());
    }

    private function isNotExemptFromFees(int $productId): bool
    {
        return $this->productDetails->get($productId)->isNotExemptFromFees();
    }

    private function hasDisallowedBlockers(int $productId): bool
    {
        $blockerIds = $this->blockers->getBlockerIds($productId);
        return !empty(array_intersect($blockerIds, [2006, 2017]));
    }

    private function isGuestCheckout(int $customerId): bool
    {
        return $this->orderPurchase->getCustomer($customerId)->isGuestCheckout();
    }
}
```

This is one focused class with one responsibility: answering "can this order be paid?".

### A Guard

Guards are specifications that throw rather than return. Use them for pre-flight checks that should hard-block an operation:

```php
interface GuardInterface
{
    /** @throws GuardException */
    public function handle(int $productId): void;
}

final class OrderCreationGuard implements GuardInterface
{
    public function __construct(
        private readonly FinanceCategorySpecification $financeCategory,
        private readonly PaymentStatusSpecification   $paymentStatus,
    ) {}

    public function handle(int $productId): void
    {
        // Both conditions must be true to block - this is a composed rule
        if ($this->financeCategory->satisfiedBy($productId)
            && $this->paymentStatus->satisfiedBy($productId)
        ) {
            throw new GuardException('Cannot create a core order for a high-value order with active dispute.');
        }
    }
}
```

The caller doesn't handle a return value - it either proceeds or catches `GuardException`:

```php
$this->orderCreationGuard->handle($productId); // throws or is silent
$order = $this->orderFactory->create($request);
```

### Composing Specifications in a Service

The service becomes a clear sequence of checks:

```php
class OrderPaymentService
{
    public function __construct(
        private readonly OrderPaymentReadySpecification $paymentReady,
        private readonly OrderAmountSpecification       $amountValid,
        private readonly OrderCreationGuard             $guard,
    ) {}

    public function initiatePayment(Order $order): void
    {
        $this->guard->handle($order->getProductId());

        if (!$this->paymentReady->isSatisfied($order)) {
            throw new OrderNotReadyException('Order is not ready for payment.');
        }

        if (!$this->amountValid->isSatisfied($order)) {
            throw new InvalidAmountException('Order amount is invalid.');
        }

        // ... process payment
    }
}
```

Reading this method, you immediately know: guard runs first (throws on hard block), then two specifications are checked in sequence.

### Testing in Isolation

Each specification is independently testable with a minimal mock setup:

```php
class OrderPaymentReadySpecificationTest extends TestCase
{
    public function testDiscountedOrderWithNoBlockersIsSatisfied(): void
    {
        $order = $this->buildDiscountedOrder();
        $spec  = new OrderPaymentReadySpecification(
            $this->buildProductDetailsRepo(exemptFromFees: true),
            $this->buildOrderPurchaseRepo(guestCheckout: false),
            $this->buildBlockersRepo(blockers: []),
        );

        $this->assertTrue($spec->isSatisfied($order));
    }

    public function testRegularOrderWithDisallowedBlockerIsNotSatisfied(): void
    {
        $order = $this->buildDigitalProductOrder();
        $spec  = new OrderPaymentReadySpecification(
            $this->buildProductDetailsRepo(exemptFromFees: false),
            $this->buildOrderPurchaseRepo(guestCheckout: false),
            $this->buildBlockersRepo(blockers: [2006]),
        );

        $this->assertFalse($spec->isSatisfied($order));
    }
}
```

No service container needed - just build the specification with its direct dependencies.

### Key Takeaways

| Design choice | Why |
|---|---|
| Specification per rule | One class, one responsibility, one test class |
| `isSatisfied()` returning bool | No exceptions in normal validation flow; caller decides what to do |
| Guard throwing exception | For pre-conditions that signal a programming error or must hard-stop |
| Composition in the service | Caller reads like a checklist; rules are independent |

**Trade-off:** With 10+ specifications in a service constructor, Symfony DI still handles it cleanly. The cost is more files. The benefit is that each rule can be changed, replaced, or disabled without touching unrelated logic. If you find yourself adding `&&` or `||` inside a specification, consider splitting it into two and composing them at the service level.
