---
layout: default
title: API Documentation with Swagger/OpenAPI
parent: Microservices
nav_order: 5
---

## API Documentation with Swagger/[OpenAPI](https://spec.openapis.org/oas/latest.html) Annotations

**Context:** Maintaining a separate API doc that drifts from the actual code is a liability. Annotating endpoints directly in the controller keeps documentation co-located with the implementation - when the code changes, the doc reminder is right there.

### Annotated Controller (PHP with OpenAPI Attributes)

```php
use OpenApi\Attributes as OA;

class OrderController
{
    #[OA\Get(
        path: '/api/v1/orders/{orderId}',
        summary: 'Retrieve a single order by ID',
        tags: ['Orders'],
        parameters: [
            new OA\Parameter(
                name: 'orderId',
                in: 'path',
                required: true,
                schema: new OA\Schema(type: 'string', format: 'uuid')
            )
        ],
        responses: [
            new OA\Response(
                response: 200,
                description: 'Order details',
                content: new OA\JsonContent(ref: '#/components/schemas/OrderResponse')
            ),
            new OA\Response(response: 404, description: 'Order not found')
        ]
    )]
    public function getOrder(string $orderId): JsonResponse
    {
        // ...
    }
}
```

### Generating the Spec

```bash
# Generate OpenAPI JSON from annotations
./vendor/bin/openapi src/ --output public/api-docs/openapi.json

# Serve with Swagger UI (https://swagger.io/tools/swagger-ui/) - Docker, for local dev
docker run -p 8082:8080 \
  -e SWAGGER_JSON=/spec/openapi.json \
  -v $(pwd)/public/api-docs:/spec \
  swaggerapi/swagger-ui
```

> **Gotcha:** Annotations only document what you write - they don't validate runtime behavior. If your annotation says the response is `200` with an `OrderResponse` schema but your code actually returns a different structure on edge cases, the doc becomes a lie. Pair annotations with response serialization tests that assert the actual output matches the documented schema.
