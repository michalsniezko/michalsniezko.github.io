---
layout: default
title: Processor Chain with Error Accumulation and Circuit Breakers
parent: PHP Patterns in Practice
nav_order: 6
---

## Processor Chain with Error Accumulation and Circuit Breakers

**Problem:** The classic Chain of Responsibility pattern stops at the first failure. In a multi-step business process (e.g. an "unbooking" workflow that reverses a sale), you need to either: (a) collect *all* errors before deciding what to do, or (b) hard-stop when a critical precondition fails. You need both behaviors from the same chain.

**Solution:** Extend the chain with an error collection DTO passed through every processor. Each processor appends errors to the collection without throwing. Hard stops are signalled by throwing a typed exception that the chain's caller catches and handles. Soft stops (pause, retry later) use a different exception type.

### The Interface

```php
interface ProcessorInterface
{
    /**
     * @throws ProcessBlockedException
     * @throws ProcessPausedException
     */
    public function process(RequestDTO $request, ErrorCollection $errors): void;
}
```

Two distinct exceptions signal two distinct recovery paths:

- `ProcessBlockedException` - the operation must not continue (e.g. payment already collected). Log and fail hard.
- `ProcessPausedException` - the operation should be retried later (e.g. an upstream service timed out).

### The Chain

```php
class ProcessorChain
{
    /** @var ProcessorInterface[] */
    private array $processors = [];

    public function __construct(private readonly LoggerInterface $logger, iterable $processors = [])
    {
        foreach ($processors as $processor) {
            $this->processors[] = $processor;
        }
    }

    /**
     * @throws ProcessBlockedException
     * @throws ProcessPausedException
     */
    public function process(RequestDTO $request, ErrorCollection $errors): void
    {
        foreach ($this->processors as $processor) {
            $processor->process($request, $errors); // exceptions propagate up immediately
        }

        // Log all accumulated non-blocking errors after the chain completes
        foreach ($errors as $error) {
            $exception = $error->getException();
            $this->logger->error(
                sprintf('Process failed in %s', $error->getProcessorName()),
                [
                    'id'               => $request->getId(),
                    'errorMessage'     => $error->getErrorMessage(),
                    'exceptionMessage' => $exception?->getMessage() ?? 'n/a',
                ]
            );
        }
    }
}
```

Note that exceptions thrown inside any processor immediately exit the `foreach`. The error collection is for *soft* errors that don't stop the chain; exceptions are for hard stops.

### A Blocking Processor (Circuit Breaker)

```php
final readonly class BlockPaidOrderProcessor implements ProcessorInterface
{
    public function __construct(private OrderRepository $orderRepository) {}

    public function process(RequestDTO $request, ErrorCollection $errors): void
    {
        $isPaid = false;
        try {
            $order = $this->orderRepository->findActiveOrder($request->getId());
            if ($order && $order->isPaid()) {
                $isPaid = true;
                $errors->add(new ErrorDTO(
                    __CLASS__,
                    $request->getId(),
                    'Cannot reverse a paid order.',
                ));
            }
        } catch (\Throwable $e) {
            $errors->add(new ErrorDTO(__CLASS__, $request->getId(), 'Payment check failed.', $e));
        }

        if ($isPaid) {
            throw new ProcessBlockedException();
        }
    }

    public static function getDefaultPriority(): int
    {
        return 9999; // runs first - highest priority number wins
    }
}
```

The processor adds an error *and then* throws. The caller knows why the chain was blocked because the error is already in the collection before the exception escapes.

### Wiring with Symfony Tagged Services

Using `_instanceof` + `!tagged_iterator` means you never touch the container when adding a processor:

```yaml
# config/services.yaml
services:
    _instanceof:
        App\Processor\ProcessorInterface:
            tags: ['app.processor']

    App\Service\ProcessorChain:
        arguments:
            $processors: !tagged_iterator { tag: 'app.processor', default_priority_method: 'getDefaultPriority' }
```

Add a new processor class that implements `ProcessorInterface` and optionally defines `getDefaultPriority()` - Symfony picks it up automatically.

### Multiple Chains from the Same Processors

The real power: you can create multiple named `ProcessorChain` instances each fed a different tag, keeping the processor classes completely reusable:

```yaml
app.processor.chain.reversal:
    class: App\Service\ProcessorChain
    arguments:
        $processors: !tagged_iterator app.processor.reversal

app.processor.chain.cancellation:
    class: App\Service\ProcessorChain
    arguments:
        $processors: !tagged_iterator app.processor.cancellation
```

Processors that are shared across flows implement multiple marker interfaces or are tagged manually.

### Key Takeaways

| Design choice | Why |
|---|---|
| Error collection DTO passed by reference | Processors accumulate soft failures without coupling to each other |
| Two exception types for two stop modes | Callers can distinguish "fail permanently" from "retry later" |
| `getDefaultPriority()` static method | Priority lives with the processor class, not in YAML |
| `_instanceof` auto-tagging | Zero wiring for new processors - just implement the interface |

**Trade-off:** Because the chain iterates all processors before logging errors, a processor that throws early skips subsequent ones. Design your blocking processors to run first (high priority number) so they gate the rest.
