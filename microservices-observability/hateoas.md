---
layout: default
title: HATEOAS
parent: Microservices & Service Design
nav_order: 9
---

## HATEOAS: Hypermedia As The Engine Of Application State

**Problem:** A client integrates with your API. Six months later you add a new transition: shipments can now be `recalled` after dispatch. The client breaks because it hardcoded the URL `/shipments/{id}/recall` based on a conversation in a Slack thread. You have to version the endpoint, update documentation, email every consumer, and still wait for them to deploy.

HATEOAS is the REST constraint that prevents this. Instead of the client knowing URLs in advance, the server embeds the available actions in every response. The client follows links. URLs can change; the link relation names stay stable.

---

### What a HATEOAS Response Looks Like

A plain REST response:

```json
{
  "id": "s-789",
  "origin": "Warsaw",
  "destination": "Berlin",
  "status": "dispatched"
}
```

The same response with hypermedia controls:

```json
{
  "id": "s-789",
  "origin": "Warsaw",
  "destination": "Berlin",
  "status": "dispatched",
  "_links": {
    "self":   { "href": "/shipments/s-789" },
    "track":  { "href": "/shipments/s-789/tracking" },
    "recall": { "href": "/shipments/s-789/recall", "method": "POST" }
  }
}
```

The `_links` block tells the client what it can do right now, given the current state. A `delivered` shipment would not include a `recall` link. The client does not need to know the URL structure or the business rules; it just checks whether the link is present.

---

### The Richardson Maturity Model

HATEOAS sits at the top of a four-level scale for REST API maturity:

| Level | Name | What it means |
|-------|------|---------------|
| 0 | The Swamp of POX | One endpoint, one HTTP method, RPC-style |
| 1 | Resources | Separate URLs per resource |
| 2 | HTTP Verbs | GET / POST / PUT / DELETE used correctly |
| 3 | Hypermedia Controls | Responses include links to available actions |

Most APIs marketed as "REST" are at Level 2. Level 3 is what Fielding's original dissertation actually described.

---

### Hypermedia Formats

The `_links` convention above is [HAL (Hypertext Application Language)](https://stateless.group/hal_specification.html), the most common format. Others exist:

| Format | Media Type | Notes |
|--------|-----------|-------|
| HAL | `application/hal+json` | `_links` and `_embedded`, simple, widely supported |
| JSON:API | `application/vnd.api+json` | Opinionated, includes pagination and relationships |
| Siren | `application/vnd.siren+json` | Adds typed `actions` with fields, closer to HTML forms |
| Collection+JSON | `application/vnd.collection+json` | List-first design |

HAL is the practical default for new PHP APIs.

---

### Implementing HAL in Symfony

#### The Resource Class

```php
final class ShipmentResource
{
    public function __construct(
        public readonly string $id,
        public readonly string $origin,
        public readonly string $destination,
        public readonly string $status,
        /** @var array<string, array{href: string, method?: string}> */
        public readonly array $links = [],
    ) {}
}
```

#### The Assembler

The assembler knows the domain rules for which links apply to which states. It is the only place that encodes "a dispatched shipment can be recalled":

```php
final class ShipmentResourceAssembler
{
    public function toResource(Shipment $shipment): ShipmentResource
    {
        $id    = $shipment->getId();
        $links = [
            'self' => ['href' => "/shipments/{$id}"],
        ];

        match ($shipment->getStatus()) {
            'pending'    => $links += [
                'dispatch' => ['href' => "/shipments/{$id}/dispatch", 'method' => 'POST'],
                'cancel'   => ['href' => "/shipments/{$id}/cancel",   'method' => 'POST'],
            ],
            'dispatched' => $links += [
                'track'  => ['href' => "/shipments/{$id}/tracking"],
                'recall' => ['href' => "/shipments/{$id}/recall", 'method' => 'POST'],
            ],
            'delivered'  => $links += [
                'track' => ['href' => "/shipments/{$id}/tracking"],
            ],
            default => [],
        };

        return new ShipmentResource(
            id:          $id,
            origin:      $shipment->getOrigin(),
            destination: $shipment->getDestination(),
            status:      $shipment->getStatus(),
            links:       $links,
        );
    }
}
```

#### The Controller

```php
#[Route('/shipments/{id}', methods: ['GET'])]
public function show(string $id): JsonResponse
{
    $shipment = $this->shipmentRepository->find($id);

    if ($shipment === null) {
        return new JsonResponse(['error' => 'not_found'], 404);
    }

    $resource = $this->assembler->toResource($shipment);

    return new JsonResponse(
        [
            'id'          => $resource->id,
            'origin'      => $resource->origin,
            'destination' => $resource->destination,
            'status'      => $resource->status,
            '_links'      => $resource->links,
        ],
        200,
        ['Content-Type' => 'application/hal+json'],
    );
}
```

---

### Collections and Pagination

HAL handles lists via `_embedded` and pagination links:

```json
{
  "_links": {
    "self":  { "href": "/shipments?page=2&limit=20" },
    "first": { "href": "/shipments?page=1&limit=20" },
    "prev":  { "href": "/shipments?page=1&limit=20" },
    "next":  { "href": "/shipments?page=3&limit=20" },
    "last":  { "href": "/shipments?page=12&limit=20" }
  },
  "total": 234,
  "page":  2,
  "_embedded": {
    "shipments": [
      {
        "id": "s-789",
        "status": "dispatched",
        "_links": {
          "self":   { "href": "/shipments/s-789" },
          "recall": { "href": "/shipments/s-789/recall", "method": "POST" }
        }
      }
    ]
  }
}
```

```php
final class ShipmentCollectionAssembler
{
    public function __construct(private readonly ShipmentResourceAssembler $itemAssembler) {}

    /** @param Shipment[] $shipments */
    public function toCollection(array $shipments, int $page, int $limit, int $total): array
    {
        $lastPage = (int) ceil($total / $limit);

        $links = [
            'self'  => ['href' => "/shipments?page={$page}&limit={$limit}"],
            'first' => ['href' => "/shipments?page=1&limit={$limit}"],
            'last'  => ['href' => "/shipments?page={$lastPage}&limit={$limit}"],
        ];

        if ($page > 1) {
            $links['prev'] = ['href' => '/shipments?page=' . ($page - 1) . "&limit={$limit}"];
        }

        if ($page < $lastPage) {
            $links['next'] = ['href' => '/shipments?page=' . ($page + 1) . "&limit={$limit}"];
        }

        return [
            '_links'    => $links,
            'total'     => $total,
            'page'      => $page,
            '_embedded' => [
                'shipments' => array_map(
                    fn(Shipment $s) => $this->flatten($this->itemAssembler->toResource($s)),
                    $shipments,
                ),
            ],
        ];
    }

    private function flatten(ShipmentResource $resource): array
    {
        return [
            'id'          => $resource->id,
            'origin'      => $resource->origin,
            'destination' => $resource->destination,
            'status'      => $resource->status,
            '_links'      => $resource->links,
        ];
    }
}
```

---

### What the Client Does With This

A HATEOAS-aware client discovers actions rather than constructing URLs:

```typescript
async function recallShipment(shipmentUrl: string): Promise<void> {
    const res  = await fetch(shipmentUrl);
    const data = await res.json();

    const recallLink = data._links?.recall;

    if (!recallLink) {
        throw new Error('Recall is not available for this shipment.');
    }

    await fetch(recallLink.href, { method: recallLink.method ?? 'POST' });
}
```

The client never constructs `/shipments/s-789/recall` from a template. It reads the URL from the response. When the server moves that URL to `/logistics/shipments/s-789/actions/recall`, the client works without changes.

---

### When HATEOAS Is Worth It

HATEOAS adds serialization cost and assembler code. It is worth that cost when:

- **The API is consumed by multiple independent clients** that you cannot coordinate deploys with. Public APIs, partner integrations, mobile apps.
- **The resource has a meaningful state machine.** A shipment with six states and conditional transitions is a good candidate. A static `GET /config` endpoint is not.
- **You want the API to be self-documenting at runtime.** A client can walk the API from the entry point and discover everything, which is useful for developer tools and generated SDKs.

It is probably not worth it for:

- Internal service-to-service calls where both sides deploy together.
- Simple CRUD APIs with no state transitions.
- Teams that will not maintain the assembler discipline over time.

---

### Entry Point

A fully HATEOAS-compliant API has a single well-known entry point. Everything else is discovered from there:

```json
GET /api

{
  "_links": {
    "shipments":  { "href": "/shipments" },
    "products":   { "href": "/products" },
    "orders":     { "href": "/orders" }
  }
}
```

Clients bookmark `/api`. They never bookmark `/shipments/s-789/recall`.

---

> **Gotcha:** Including links is not enough on its own. If your API documentation says "to recall a shipment, POST to `/shipments/{id}/recall`" and clients read the docs instead of the responses, you have added `_links` to your JSON with no actual decoupling. HATEOAS only delivers its benefit if clients are written to follow links. Document the link relation names (`self`, `recall`, `track`), not the URLs.

The link relation names and the response schemas for each state can be formally described using [API Documentation with Swagger/OpenAPI](swagger-openapi.md), giving clients a static contract alongside the runtime self-description that HATEOAS provides.

---

### For AI agents

```
For REST APIs that evolve independently of clients: embed available action links in responses using HAL format (_links). Clients discover endpoints and state transitions from the response instead of hardcoding URLs. Never version URLs if state transitions can be expressed as links.
```

Reference: `https://michalsniezko.github.io/microservices-observability/hateoas.html`
