---
layout: default
title: JS Testing with Jest
parent: Testing, Concurrency, Distributed Locks
nav_order: 5
---

## JS Testing with Jest

**Scenario:** Your frontend calculates a vehicle's monthly payment based on price, interest rate, and loan term. This logic lives in a pure TypeScript module - no DOM, no API calls. You need fast, isolated tests with good mocking support for the edge cases (zero interest, negative values, rounding).

### Module Under Test

```typescript
// src/finance/calculator.ts
export function monthlyPayment(
    principal: number,
    annualRate: number,
    termMonths: number
): number {
    if (termMonths <= 0) throw new Error('Term must be positive');
    if (annualRate === 0) return parseFloat((principal / termMonths).toFixed(2));

    const monthlyRate = annualRate / 12 / 100;
    const factor = Math.pow(1 + monthlyRate, termMonths);
    return parseFloat(((principal * monthlyRate * factor) / (factor - 1)).toFixed(2));
}
```

### Jest Test

```typescript
// src/finance/__tests__/calculator.test.ts
import { monthlyPayment } from '../calculator';

describe('monthlyPayment', () => {
    it('calculates standard loan payment', () => {
        // $20,000 at 5% for 60 months
        expect(monthlyPayment(20000, 5, 60)).toBe(377.42);
    });

    it('handles zero interest', () => {
        expect(monthlyPayment(12000, 0, 12)).toBe(1000.00);
    });

    it('throws on zero term', () => {
        expect(() => monthlyPayment(10000, 5, 0)).toThrow('Term must be positive');
    });
});
```

### Mocking an API Module

```typescript
// src/finance/priceService.ts
export async function fetchVehiclePrice(vehicleId: string): Promise<number> {
    const res = await fetch(`/api/v1/vehicles/${vehicleId}/price`);
    return (await res.json()).price;
}

// src/finance/__tests__/priceService.test.ts
import { fetchVehiclePrice } from '../priceService';

// Mock the global fetch
global.fetch = jest.fn();

describe('fetchVehiclePrice', () => {
    it('returns price from API', async () => {
        (fetch as jest.Mock).mockResolvedValueOnce({
            json: async () => ({ price: 25000 }),
        });

        const price = await fetchVehiclePrice('v-123');
        expect(price).toBe(25000);
        expect(fetch).toHaveBeenCalledWith('/api/v1/vehicles/v-123/price');
    });

    it('propagates network errors', async () => {
        (fetch as jest.Mock).mockRejectedValueOnce(new Error('Network failure'));

        await expect(fetchVehiclePrice('v-123')).rejects.toThrow('Network failure');
    });
});
```

> **Safety First:** Jest runs tests in parallel by default (one worker per CPU core). If your tests share mutable state (e.g., `global.fetch = jest.fn()` without cleanup), tests can leak state into each other. Always restore mocks in `afterEach` or use `jest.restoreAllMocks()`. For true isolation, use `--runInBand` to run sequentially - slower, but eliminates parallel flakiness during debugging.
