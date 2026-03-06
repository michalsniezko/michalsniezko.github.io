---
layout: default
title: Prophecy & Promise Pattern (PHPUnit)
parent: Backend Patterns & Optimization for High-Volume Systems
nav_order: 3
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
