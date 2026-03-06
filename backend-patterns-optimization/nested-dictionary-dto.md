---
layout: default
title: Nested Dictionary DTOs
parent: Backend Patterns & Optimization for High-Volume Systems
nav_order: 6
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
