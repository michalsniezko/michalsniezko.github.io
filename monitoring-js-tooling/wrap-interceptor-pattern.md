---
layout: default
title: Wrap Interceptor Pattern (Node.js Lambda)
parent: Monitoring with TICK Stack, JavaScript Tooling
nav_order: 5
---

## Wrap Interceptor Pattern (Node.js Lambda)

**Use Case:** Every Lambda handler needs the same boilerplate: parse the event, log the request, catch errors, format the response, log the duration. Copy-pasting this into 30 handlers means 30 places to update when you change the error format or add a new header.

**Solution:** A higher-order function that wraps any handler with cross-cutting concerns. The business logic handler stays pure - it receives parsed input and returns data. The wrapper handles HTTP plumbing.

### The Interceptor

```javascript
// lib/interceptor.js

function wrapHandler(businessLogic, options = {}) {
    const { requireAuth = true } = options;

    return async (event, context) => {
        const start = Date.now();
        const requestId = context.awsRequestId;

        console.log(JSON.stringify({
            event: 'request_received',
            requestId,
            path: event.path,
            method: event.httpMethod,
            // Never log event.body in production - may contain PII
        }));

        try {
            // Auth check (if required)
            if (requireAuth && !event.headers?.Authorization) {
                return response(401, { error: 'Missing authorization header' });
            }

            // Parse body safely
            const body = event.body ? JSON.parse(event.body) : null;

            // Call the actual business logic
            const result = await businessLogic({ body, event, context });

            console.log(JSON.stringify({
                event: 'request_completed',
                requestId,
                durationMs: Date.now() - start,
                statusCode: 200,
            }));

            return response(200, result);

        } catch (error) {
            console.error(JSON.stringify({
                event: 'request_failed',
                requestId,
                durationMs: Date.now() - start,
                error: error.message,
                stack: error.stack,
            }));

            if (error.name === 'ValidationError') {
                return response(400, { error: error.message });
            }

            return response(500, { error: 'Internal server error' });
        }
    };
}

function response(statusCode, body) {
    return {
        statusCode,
        headers: {
            'Content-Type': 'application/json',
            'X-Request-Id': body?.requestId || 'unknown',
        },
        body: JSON.stringify(body),
    };
}

module.exports = { wrapHandler };
```

### Clean Business Logic Handlers

```javascript
// handlers/createOrder.js
const { wrapHandler } = require('../lib/interceptor');
const { OrderService } = require('../core/orderService');

const service = new OrderService();  // reused across warm invocations

module.exports.handler = wrapHandler(async ({ body }) => {
    // Pure business logic - no HTTP plumbing
    const order = await service.create({
        productId: body.productId,
        quantity: body.quantity,
    });

    return { orderId: order.id, status: order.status };
});

// handlers/healthCheck.js
const { wrapHandler } = require('../lib/interceptor');

module.exports.handler = wrapHandler(
    async () => ({ status: 'ok', timestamp: new Date().toISOString() }),
    { requireAuth: false }
);
```

> **Clean Code Tip:** Resist the urge to make the interceptor configurable for every edge case (custom serializers, per-route middleware chains, plugin systems). That's an HTTP framework - use Express or Fastify at that point. The wrapper should handle 3 things: logging, error formatting, and response shaping. Anything more complex means your Lambda is doing too much.
