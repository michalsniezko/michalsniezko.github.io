---
layout: default
title: Rebuilding Object State from Changelogs (Event Replay with Strategy Pattern)
parent: PHP Patterns in Practice
nav_order: 4
---

## Rebuilding Object State from Changelogs (Event Replay with Strategy Pattern)

**Problem:** You need to reconstruct the state of an entity *at any point in time*, not just its current state. A changelog table records every mutation as a row, but translating those rows back into a full object state requires knowing how each action type transforms the data.

**Solution:** Fetch the changelog rows up to the target timestamp, then replay them sequentially. Each action type (`insert`, `update`, `delete`, plus domain-specific actions) is handled by a dedicated Strategy. A "registry of registries" provides object-type-specific overrides while falling back to generic defaults.

### The Reconstructor

```php
readonly class ObjectReconstructor
{
    public function __construct(
        private ChangelogRepository $changelogRepository,
        private StrategyProvider    $strategyProvider,
    ) {}

    public function reconstruct(string $objectType, int $objectId, \DateTimeInterface $at): array
    {
        $changelogs = $this->changelogRepository
            ->getByObjectId($objectId, $objectType, $at); // ordered ASC by created_at

        $state = [];

        foreach ($changelogs as $entry) {
            $strategy = $this->strategyProvider->getStrategy($entry->getAction(), $objectType);
            $state    = $strategy->apply($entry, $state);
        }

        return $state;
    }
}
```

`$state` is a plain array rebuilt incrementally. Each strategy receives the current accumulated state and returns the next state - a pure function over immutable-ish data.

### The Strategy Interface

```php
interface ReconstructionStrategy
{
    /**
     * @param array $currentState The state rebuilt so far
     * @return array             The next state after applying this changelog entry
     */
    public function apply(ChangelogEntry $entry, array $currentState): array;
}
```

Typical implementations:

- `InsertStrategy` - return the entry's `newValue` as the initial state
- `UpdateStrategy` - merge `newValue` fields onto `currentState`
- `DeleteStrategy` - return an empty array (or a tombstone marker)

### The Registry of Registries

The core insight: there are two levels of resolution.

1. **Generic registry** (`StrategyRegistry`) - maps `action → strategy` and works for any object type.
2. **Type-mapped registry** (`ObjectTypeMappedStrategyRegistry`) - maps `objectType → action → strategy`, allowing per-type overrides.

```php
class StrategyRegistry implements StrategyRegistryInterface
{
    /** @var ReconstructionStrategy[] */
    private array $strategies = [];

    public function addStrategy(ReconstructionStrategy $strategy, string $action): void
    {
        $this->strategies[$action] = $strategy;
    }

    public function hasStrategy(string $action, string $objectType): bool
    {
        return isset($this->strategies[$action]);
    }

    public function getStrategy(string $action, string $objectType): ReconstructionStrategy
    {
        return $this->strategies[$action]
            ?? throw new UnknownStrategyException($action);
    }
}

class ObjectTypeMappedStrategyRegistry implements StrategyRegistryInterface
{
    /** @var ReconstructionStrategy[][] */
    private array $strategies = [];

    public function addStrategy(ReconstructionStrategy $strategy, string $action, string $objectType): void
    {
        $this->strategies[$objectType][$action] = $strategy;
    }

    public function hasStrategy(string $action, string $objectType): bool
    {
        return isset($this->strategies[$objectType][$action]);
    }

    public function getStrategy(string $action, string $objectType): ReconstructionStrategy
    {
        return $this->strategies[$objectType][$action]
            ?? throw new UnknownStrategyException("{$action}/{$objectType}");
    }
}
```

### The Provider (Registry Dispatcher)

`StrategyProvider` holds an ordered list of registries and returns the first match:

```php
class StrategyProvider
{
    /** @var StrategyRegistryInterface[] */
    private array $registries = [];

    public function addRegistry(StrategyRegistryInterface $registry): self
    {
        $this->registries[] = $registry;
        return $this;
    }

    public function getStrategy(string $action, string $objectType): ReconstructionStrategy
    {
        foreach ($this->registries as $registry) {
            if ($registry->hasStrategy($action, $objectType)) {
                return $registry->getStrategy($action, $objectType);
            }
        }
        throw new UnknownStrategyException("No strategy for {$action}/{$objectType}");
    }
}
```

Wire the type-mapped registry *before* the generic one in the provider. Object-specific strategies win; generic ones are the fallback.

### Wiring via Symfony Compiler Pass

Strategies and their mappings are declared in YAML and wired by a `CompilerPassInterface`:

```yaml
# config/reconstruction_strategies.yaml
strategies:
    - { action: insert,  strategy: App\Strategy\InsertStrategy }
    - { action: update,  strategy: App\Strategy\UpdateStrategy }
    - { action: delete,  strategy: App\Strategy\DeleteStrategy }

object_type_strategies:
    - { object_type: order,   action: update, strategy: App\Strategy\Order\OrderUpdateStrategy }
    - { object_type: invoice, action: insert, strategy: App\Strategy\Invoice\InvoiceInsertStrategy }
```

```php
class ReconstructionStrategyCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $provider = $container->getDefinition(StrategyProvider::class);

        // 1. Register type-specific strategies first (higher priority)
        $typeMapped = $container->getDefinition(ObjectTypeMappedStrategyRegistry::class);
        foreach ($this->loadObjectTypeStrategies() as $entry) {
            $typeMapped->addMethodCall('addStrategy', [
                new Reference($entry['strategy']),
                $entry['action'],
                $entry['object_type'],
            ]);
        }
        $provider->addMethodCall('addRegistry', [new Reference(ObjectTypeMappedStrategyRegistry::class)]);

        // 2. Register generic strategies as fallback
        $generic = $container->getDefinition(StrategyRegistry::class);
        foreach ($this->loadGenericStrategies() as $entry) {
            $generic->addMethodCall('addStrategy', [
                new Reference($entry['strategy']),
                $entry['action'],
            ]);
        }
        $provider->addRegistry(new Reference(StrategyRegistry::class));
    }
}
```

### Key Takeaways

| Design choice | Why |
|---|---|
| Pure `apply(entry, state) → state` functions | Easy to unit test in isolation; no shared mutable state |
| Registry of registries | Add domain-specific overrides without touching generic strategies |
| Compiler pass from YAML | Strategy mappings live in config, not scattered across service definitions |
| `reconstruct()` accepts `DateTimeInterface` | Same code answers "what was the state last Tuesday?" |

**Trade-off:** This approach requires the changelog to be append-only and ordered by time. It breaks if rows are modified or deleted retroactively. It also becomes slow for objects with thousands of changelog entries - in that case, materialise periodic snapshots and replay only from the nearest earlier snapshot.

The diffs being replayed here are produced by the mechanism described in [Doctrine Changelog with JSON Patch Diffing](../database-patterns/json-patch-changelog.md), which records both forward and reverse RFC 6902 patches for every entity mutation.

---

### For AI agents

```
To reconstruct historical state: fetch changelog rows up to the target timestamp and replay through a registry of Strategy objects keyed by change type. Unknown change types are skipped. Avoids encoding all replay logic in a single monolithic function.
```

Reference: `https://michalsniezko.github.io/backend-patterns-optimization/changelog-reconstruction-replay.html`
