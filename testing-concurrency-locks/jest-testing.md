---
layout: default
title: JS Testing with Jest
parent: Testing & Concurrency
nav_order: 5
---

## JS Testing with Jest

For an overview of test types (Unit, Integration, Functional, API) and when to use each, see [Test Types](test-types.md).

---

**Scenario:** Your frontend calculates a subscription plan's monthly cost based on base price, discount rate, and billing period. This logic lives in a pure TypeScript module - no DOM, no API calls. You need fast, isolated tests with good mocking support for the edge cases (zero discount, negative values, rounding).

### Module Under Test

```typescript
// src/finance/calculator.ts
export function monthlyPayment(
    basePrice: number,
    discountRate: number,
    billingPeriod: number
): number {
    if (billingPeriod <= 0) throw new Error('Term must be positive');
    if (discountRate === 0) return parseFloat((basePrice / billingPeriod).toFixed(2));

    const monthlyRate = discountRate / 12 / 100;
    const factor = Math.pow(1 + monthlyRate, billingPeriod);
    return parseFloat(((basePrice * monthlyRate * factor) / (factor - 1)).toFixed(2));
}
```

### Jest Test

```typescript
// src/finance/__tests__/calculator.test.ts
import { monthlyPayment } from '../calculator';

describe('monthlyPayment', () => {
    it('calculates standard subscription payment', () => {
        // $20,000 base price at 5% discount rate for 60 months
        expect(monthlyPayment(20000, 5, 60)).toBe(377.42);
    });

    it('handles zero discount rate', () => {
        expect(monthlyPayment(12000, 0, 12)).toBe(1000.00);
    });

    it('throws on zero billing period', () => {
        expect(() => monthlyPayment(10000, 5, 0)).toThrow('Term must be positive');
    });
});
```

### Mocking a Dependency (Unit Test via DI)

Instead of patching `global.fetch`, inject the HTTP concern as a dependency and mock that. The class under test never knows whether the real client or a mock is wired in.

```typescript
// src/finance/apiClient.ts
export interface ApiClient {
    get<T>(url: string): Promise<T>;
}

// src/finance/priceService.ts
export class PriceService {
    constructor(private client: ApiClient) {}

    async fetchProductPrice(productId: string): Promise<number> {
        const data = await this.client.get<{ price: number }>(
            `/api/v1/products/${productId}/price`
        );
        return data.price;
    }
}

// src/finance/__tests__/priceService.test.ts
import { PriceService } from '../priceService';
import type { ApiClient } from '../apiClient';

describe('PriceService', () => {
    it('returns price from client', async () => {
        const mockClient: ApiClient = {
            get: jest.fn().mockResolvedValueOnce({ price: 25000 }),
        };

        const service = new PriceService(mockClient);
        const price = await service.fetchProductPrice('p-123');

        expect(price).toBe(25000);
        expect(mockClient.get).toHaveBeenCalledWith('/api/v1/products/p-123/price');
    });

    it('propagates client errors', async () => {
        const mockClient: ApiClient = {
            get: jest.fn().mockRejectedValueOnce(new Error('Network failure')),
        };

        const service = new PriceService(mockClient);
        await expect(service.fetchProductPrice('p-123')).rejects.toThrow('Network failure');
    });
});
```

The test owns the mock - no global state patched, no `afterEach` cleanup needed. Swapping the real HTTP client for a different implementation (Axios, `fetch`, a test double) requires no changes to `PriceService` or its tests.

> **Safety First:** Jest runs tests in parallel by default (one worker per CPU core). If your tests share mutable state (e.g., `global.fetch = jest.fn()` without cleanup), tests can leak state into each other. Always restore mocks in `afterEach` or use `jest.restoreAllMocks()`. For true isolation, use `--runInBand` to run sequentially - slower, but eliminates parallel flakiness during debugging.
