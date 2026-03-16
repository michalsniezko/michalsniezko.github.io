---
layout: default
title: Nested Dictionary DTOs
parent: PHP Patterns in Practice
nav_order: 3
---

## Nested Dictionary DTOs

**Problem:** The frontend renders dynamic filter dropdowns (product category, status, format) that need a structured list of possible values. Hardcoding these on the frontend means deploying a JS build every time a value changes. Sending raw DB enums is messy and leaks internal naming.

**Solution:** Return a single "dictionary" endpoint that nests all filterable dimensions into a typed DTO. The frontend calls it once on page load and populates all dropdowns from the response.

### DTO Structure

```php
class FilterDictionary
{
    public function __construct(
        /** @var DictionaryEntry[] */
        public readonly array $categories,
        /** @var DictionaryEntry[] */
        public readonly array $formats,
        /** @var DictionaryEntry[] */
        public readonly array $statuses,
        /** @var DictionaryEntry[] */
        public readonly array $transmissions,
    ) {}
}

class DictionaryEntry
{
    public function __construct(
        public readonly string $key,   // internal value: "physical"
        public readonly string $label, // display value: "Physical"
        public readonly ?string $group = null, // optional grouping: "Books" for categories
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
            categories: $this->buildCategories(),
            formats: [
                new DictionaryEntry('digital', 'Digital'),
                new DictionaryEntry('physical', 'Physical'),
                new DictionaryEntry('bundle', 'Bundle'),
                new DictionaryEntry('subscription', 'Subscription'),
            ],
            statuses: $this->buildStatuses($locale),
            transmissions: [
                new DictionaryEntry('manual', 'Manual'),
                new DictionaryEntry('automatic', 'Automatic'),
            ],
        );
    }

    private function buildCategories(): array
    {
        return [
            new DictionaryEntry('electronics', 'Electronics', group: 'Hardware'),
            new DictionaryEntry('accessories', 'Accessories', group: 'Hardware'),
            new DictionaryEntry('software', 'Software', group: 'Digital'),
            new DictionaryEntry('media', 'Media', group: 'Digital'),
        ];
    }

    private function buildStatuses(string $locale): array
    {
        // In production, load translations from a catalog
        return [
            new DictionaryEntry('active', $locale === 'de' ? 'Aktiv' : 'Active'),
            new DictionaryEntry('inactive', $locale === 'de' ? 'Inaktiv' : 'Inactive'),
            new DictionaryEntry('archived', $locale === 'de' ? 'Archiviert' : 'Archived'),
        ];
    }
}
```

### API Response

```json
{
  "categories": [
    {"key": "electronics", "label": "Electronics", "group": "Hardware"},
    {"key": "accessories", "label": "Accessories", "group": "Hardware"},
    {"key": "software", "label": "Software", "group": "Digital"}
  ],
  "formats": [
    {"key": "digital", "label": "Digital"},
    {"key": "physical", "label": "Physical"},
    {"key": "bundle", "label": "Bundle"}
  ],
  "statuses": [
    {"key": "active", "label": "Active"},
    {"key": "inactive", "label": "Inactive"}
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

> **Performance Tip:** Dictionary data changes rarely. Set `Cache-Control` headers aggressively (5–15 min) and consider a CDN or reverse proxy (Varnish) in front of this endpoint. For large dictionaries (10k+ catalog entries), add an `ETag` header based on a hash of the data so clients skip re-downloading unchanged payloads (`304 Not Modified`).

The TypeScript counterpart that consumes this endpoint, mirrors the DTO structure into interfaces, and populates filter dropdowns is described in [Environment Syncing: Frontend Dictionaries from Backend DTOs](../js-frontend-tooling/frontend-dictionary-sync.md).
