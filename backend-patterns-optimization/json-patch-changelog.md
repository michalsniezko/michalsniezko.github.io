---
layout: default
title: Doctrine Changelog with JSON Patch Diffing (RFC 6902)
parent: PHP Patterns in Practice
nav_order: 8
---

## Doctrine Changelog with JSON Patch Diffing (RFC 6902)

**Problem:** You need a full audit trail of entity changes. For most scalar fields a before/after value is enough. But for JSON columns storing structured data (config arrays, feature flags, nested objects), storing the full JSON snapshot on every change is wasteful - a one-field tweak stores hundreds of bytes you don't need.

**Solution:** Use a Doctrine EventSubscriber to intercept every entity change. For scalar fields, store the old and new value directly. For JSON fields, compute an [RFC 6902 JSON Patch](https://datatracker.ietf.org/doc/html/rfc6902) diff - a compact, machine-readable description of *what changed*, not the whole new state. Store both the forward patch (old→new) and the reverse patch (new→old) so any change can be undone.

### The Interface

Entities opt into changelog tracking by implementing a simple interface:

```php
interface ChangelogAwareInterface
{
    /** The root entity that "owns" the change (used for grouping in the changelog) */
    public function getParentObject(): self;

    /** The action names this entity should track (e.g. ['insert', 'update', 'delete']) */
    public function getChangelogActions(): array;

    public function getId(): int|string|null;

    public function getObjectType(): string;
}
```

### The EventSubscriber

```php
class ChangelogSubscriber implements EventSubscriber
{
    /** @var ChangelogEntry[] - accumulated during the current flush cycle */
    private array $pendingEntries = [];

    public function __construct(private readonly ChangelogService $changelogService) {}

    public function getSubscribedEvents(): array
    {
        return [Events::postPersist, Events::preUpdate, Events::preRemove, Events::postFlush];
    }

    public function postPersist(PostPersistEventArgs $args): void
    {
        $this->pendingEntries[] = $this->changelogService->buildInsertEntry($args);
    }

    public function preRemove(PreRemoveEventArgs $args): void
    {
        $this->pendingEntries[] = $this->changelogService->buildDeleteEntry($args);
    }

    public function preUpdate(PreUpdateEventArgs $args): void
    {
        foreach ($args->getEntityChangeSet() as $field => [$oldValue, $newValue]) {
            $entry = $this->buildUpdateEntry($args, $field, $oldValue, $newValue);
            if ($entry !== null) {
                $this->pendingEntries[] = $entry;
            }
        }
    }

    /** Flush the accumulated entries in a second pass after the main flush. */
    public function postFlush(PostFlushEventArgs $args): void
    {
        if (empty($this->pendingEntries)) {
            return;
        }
        $entries = $this->pendingEntries;
        $this->pendingEntries = [];

        $em = $args->getObjectManager();
        foreach ($entries as $entry) {
            $em->persist($entry);
        }
        $em->flush(); // triggers postFlush again, but pendingEntries is empty now
    }
}
```

### JSON Patch Diffing

The interesting part is inside `buildUpdateEntry`. For JSON columns, instead of storing the full serialized array, compute the patch:

```php
private function buildUpdateEntry(
    PreUpdateEventArgs $args,
    string $field,
    mixed $oldValue,
    mixed $newValue,
): ?ChangelogEntry {
    // Normalize objects (relations, DateTimes) to their scalar equivalents
    $oldValue = $this->normalize($args, $field, $oldValue);
    $newValue = $this->normalize($args, $field, $newValue);

    if ($oldValue === $newValue) {
        return null; // No actual change after normalization
    }

    $forwardPatch = null;
    $reversePatch = null;

    // For JSON columns where both sides are non-empty arrays, compute RFC 6902 diffs
    $fieldType = $this->getFieldType($args, $field);
    if ($fieldType === Types::JSON
        && is_array($oldValue) && !empty($oldValue)
        && is_array($newValue) && !empty($newValue)
    ) {
        $diff = new JsonDiff($oldValue, $newValue, JsonDiff::TOLERATE_ASSOCIATIVE_ARRAYS);
        $forwardPatch = json_encode($diff->getPatch()->jsonSerialize());

        $reverseDiff = new JsonDiff($newValue, $oldValue, JsonDiff::TOLERATE_ASSOCIATIVE_ARRAYS);
        $reversePatch = json_encode($reverseDiff->getPatch()->jsonSerialize());

        // Store the patches as the old/new values instead of the full JSON
        $oldValue = $reversePatch;
        $newValue = $forwardPatch;
    }

    // Fall back to full JSON for array→empty or empty→array transitions
    if (is_array($oldValue)) {
        $oldValue = json_encode($oldValue);
    }
    if (is_array($newValue)) {
        $newValue = json_encode($newValue);
    }

    return $this->changelogService->buildEntry($args, $field, (string)$oldValue, (string)$newValue);
}
```

### What a JSON Patch Looks Like

Given a configuration field that changed:

```json
// Old value
{"theme": "light", "notifications": {"email": true, "sms": false}, "pageSize": 20}

// New value
{"theme": "light", "notifications": {"email": true, "sms": true}, "pageSize": 50}
```

Forward patch (stored as `new_value` in changelog):
```json
[
  {"op": "replace", "path": "/notifications/sms", "value": true},
  {"op": "replace", "path": "/pageSize",           "value": 50}
]
```

Reverse patch (stored as `old_value`):
```json
[
  {"op": "replace", "path": "/notifications/sms", "value": false},
  {"op": "replace", "path": "/pageSize",           "value": 20}
]
```

Two small JSON arrays instead of two full object snapshots.

### Applying a Reverse Patch (Undo)

The reverse patch makes point-in-time rollback trivial:

```php
use Swaggest\JsonDiff\JsonPatch;

$currentState = $entity->getConfig();
$reversePatch = json_decode($changelogEntry->getOldValue(), true);

$patched = JsonPatch::import($reversePatch)->apply(json_decode(json_encode($currentState)));
$entity->setConfig((array) $patched);
```

### Key Takeaways

| Design choice | Why |
|---|---|
| RFC 6902 patches for JSON fields | Stores only the delta, not the full snapshot - much smaller for large JSON fields |
| Forward + reverse patches | Enables both audit ("what changed?") and rollback ("undo this change") |
| `postFlush` accumulation pattern | Avoids recursive flush by clearing `$pendingEntries` before the second `flush()` |
| `preUpdate` instead of `postUpdate` | The changeset is available only in `preUpdate`; `postUpdate` has no diff data |

**Trade-off:** JSON Patch computation adds CPU cost per update. For entities with large JSON fields that change frequently, this is a worthwhile trade. For entities that update hundreds of times per second or whose JSON fields are always fully replaced, storing only the new snapshot is simpler and faster.
