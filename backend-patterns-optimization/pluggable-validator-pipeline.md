---
layout: default
title: Pluggable Validator Pipeline with Symfony Tagged Services
parent: PHP Patterns in Practice
nav_order: 5
---

## Pluggable Validator Pipeline with Symfony Tagged Services

**Problem:** A booking validation service needs to run 25+ checks against three different entity types (a product, a merchant, a shipping slot). Different booking contexts (manual, auto, package-list) require different subsets of validators. Adding a new validator should not require editing the service or its wiring.

**Solution:** Define three typed validator interfaces. The validation service constructor accepts `iterable $validators` and sorts them into typed pools at construction time. Validators are registered via Symfony service tags. Multiple named instances of the same service class get different tag subsets.

### The Validator Interfaces

One interface per entity type:

```php
interface ProductValidatorInterface
{
    public function validateProduct(Product $product): ?ValidationError;
}

interface MerchantValidatorInterface
{
    public function validateMerchant(Merchant $merchant): ?ValidationError;
}

interface PackageValidatorInterface
{
    public function validatePackage(Package $package): ?ValidationError;
}
```

Each validator returns a `ValidationError` (non-null = failed) or `null` (passed). No exceptions in normal validation flow.

### The Service

The service sorts the flat `$validators` iterable into three typed pools at construction time:

```php
class BookingValidationService
{
    /** @var ProductValidatorInterface[] */
    private array $productValidators = [];

    /** @var MerchantValidatorInterface[] */
    private array $merchantValidators = [];

    /** @var PackageValidatorInterface[] */
    private array $packageValidators = [];

    public function __construct(iterable $validators = [])
    {
        foreach ($validators as $validator) {
            // A single class can implement multiple interfaces
            if ($validator instanceof ProductValidatorInterface) {
                $this->productValidators[] = $validator;
            }
            if ($validator instanceof MerchantValidatorInterface) {
                $this->merchantValidators[] = $validator;
            }
            if ($validator instanceof PackageValidatorInterface) {
                $this->packageValidators[] = $validator;
            }
        }
    }

    /** @return ValidationError[] */
    public function validate(Merchant $merchant, Package $package): array
    {
        $errors = [];

        foreach ($this->productValidators as $v) {
            if ($error = $v->validateProduct($package->getProduct())) {
                $errors[] = $error;
            }
        }

        foreach ($this->packageValidators as $v) {
            if ($error = $v->validatePackage($package)) {
                $errors[] = $error;
            }
        }

        foreach ($this->merchantValidators as $v) {
            if ($error = $v->validateMerchant($merchant)) {
                $errors[] = $error;
            }
        }

        return $errors; // empty = valid
    }
}
```

### A Validator

Each validator is a small, focused class:

```php
final class ProductIsUnavailableValidator implements ProductValidatorInterface
{
    public function validateProduct(Product $product): ?ValidationError
    {
        if ($product->isUnavailable()) {
            return new ValidationError('product-unavailable');
        }
        return null;
    }
}

final class MerchantDebtStatusValidator implements MerchantValidatorInterface
{
    public function __construct(private readonly DebtRepository $debts) {}

    public function validateMerchant(Merchant $merchant): ?ValidationError
    {
        if ($this->debts->hasOutstandingDebt($merchant->getId())) {
            return new ValidationError('merchant-has-outstanding-debt');
        }
        return null;
    }
}
```

### Wiring: Context-Specific Validator Sets

The key to flexible context-based validation is in the YAML. The same service class gets multiple named instances, each receiving a different subset of tagged validators:

```yaml
# config/services.yaml

# Tag each validator with the contexts it applies to
App\Validator\ProductIsUnavailableValidator:
    tags: ['booking.validate.auto', 'booking.validate.manual', 'booking.validate.package']

App\Validator\MerchantDebtStatusValidator:
    tags: ['booking.validate.manual']  # Only manual booking checks debt status

App\Validator\RecoveryRateTooLowValidator:
    arguments:
        $minRate: '%recovery_rate_threshold%'
    tags: ['booking.validate.auto', 'booking.validate.package']

# One instance of BookingValidationService per booking context
booking.validator.auto:
    class: App\Service\BookingValidationService
    arguments:
        $validators: !tagged_iterator booking.validate.auto

booking.validator.manual:
    class: App\Service\BookingValidationService
    arguments:
        $validators: !tagged_iterator booking.validate.manual

booking.validator.package:
    class: App\Service\BookingValidationService
    arguments:
        $validators: !tagged_iterator booking.validate.package
```

### Adding a New Validator

1. Create the class, implement the correct interface(s).
2. Add it to the services file with the appropriate tags.
3. Done. No changes to `BookingValidationService`. No changes to other validators.

```php
// New requirement: block bookings when the product hasn't passed compliance check
final class ProductComplianceRequiredValidator implements ProductValidatorInterface
{
    public function __construct(private readonly ComplianceRepository $compliance) {}

    public function validateProduct(Product $product): ?ValidationError
    {
        if (!$this->compliance->hasPassed($product->getId())) {
            return new ValidationError('compliance-check-required');
        }
        return null;
    }
}
```

```yaml
App\Validator\ProductComplianceRequiredValidator:
    tags: ['booking.validate.auto', 'booking.validate.manual']
```

### Key Takeaways

| Design choice | Why |
|---|---|
| Three typed interfaces | A validator that checks both product and merchant implements both - no duplication |
| Constructor sorting into pools | Type-safe dispatch at runtime; no `instanceof` checks during validation |
| Tag-per-context (not tag-per-type) | Context controls which validators run - a merchant check can be opt-in per context |
| Multiple named service instances | One service class, three configurations; DI container handles the rest |

**Trade-off:** Tags are invisible at the class level - you have to look in `services.yaml` to know which contexts a validator participates in. For large teams, consider adding a `#[AsTaggedItem('booking.validate.auto')]` attribute directly on the class to make it self-documenting. Symfony's `autoconfigure` with `_instanceof` can also auto-tag all classes implementing an interface, removing the need for explicit tags entirely when all validators should apply in all contexts.
