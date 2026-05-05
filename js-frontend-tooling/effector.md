---
layout: default
title: Effector
parent: JavaScript & Frontend Tooling
nav_order: 4
---

## Effector

[Effector](https://effector.dev/) is a reactive state management library for JavaScript and TypeScript. Unlike Redux (which centralises everything into one store) or MobX (which makes objects observable), Effector is built around three primitives - stores, events, and effects - that compose together through explicit data flow.

It works with React, Vue, Solid, or plain JS, and has first-class TypeScript support.

---

### The Three Primitives

**Store** (`$` prefix by convention) - holds a value. Immutable from the outside; updated only through events.

**Event** - a function that signals something happened. Can carry a payload. Think of it as a typed message.

**Effect** - an event that wraps async work (API calls, timers). Has built-in `pending`, `done`, `fail`, and `finally` derived events.

```typescript
import { createStore, createEvent, createEffect } from 'effector';

// Store: holds the current user
const $user = createStore<User | null>(null);

// Event: signals the user logged out
const loggedOut = createEvent();

// Effect: fetches a user from the API
const fetchUserFx = createEffect(async (userId: string) => {
    const res = await fetch(`/api/users/${userId}`);
    if (!res.ok) throw new Error('Failed to fetch user');
    return res.json() as Promise<User>;
});
```

By convention, stores are prefixed with `$` and effects with `Fx`. These are community conventions, not enforced by the library.

---

### Updating Stores with `.on`

The simplest way to update a store is `.on(event, reducer)`. The reducer receives the current state and the event payload, and returns the next state.

```typescript
const $counter = createStore(0);

const incremented = createEvent();
const decremented = createEvent();
const reset = createEvent();

$counter
    .on(incremented, (count) => count + 1)
    .on(decremented, (count) => count - 1)
    .reset(reset);

incremented(); // 1
incremented(); // 2
decremented(); // 1
reset();       // 0
```

Stores are never mutated directly. `incremented()` fires the event; the store reacts to it.

---

### Connecting Events and Stores with `sample`

`sample` is Effector's most powerful operator. It declaratively wires units together: "when `clock` fires, take value from `source`, optionally filter and transform it, then send to `target`."

```typescript
import { createStore, createEvent, sample } from 'effector';

const $query = createStore('');
const $results = createStore<string[]>([]);

const searchChanged = createEvent<string>();
const searchFx = createEffect(async (query: string) => {
    const res = await fetch(`/api/search?q=${query}`);
    return res.json() as Promise<string[]>;
});

// When searchChanged fires, update $query
sample({
    clock: searchChanged,
    target: $query,
});

// When $query changes, trigger search - but only if query is non-empty
sample({
    clock: $query,
    filter: (query) => query.length > 0,
    target: searchFx,
});

// When search succeeds, update $results
sample({
    clock: searchFx.doneData,
    target: $results,
});
```

`clock` - what triggers the sample (event, effect, or store update)
`source` - value to read when clock fires (optional, defaults to clock's payload)
`filter` - predicate or boolean store; skips if false
`fn` - transforms the value before sending to target
`target` - where to send the result (store, event, or effect)

---

### Effects: Handling Async Work

`createEffect` wraps async logic and exposes derived events for every lifecycle stage:

```typescript
const fetchUserFx = createEffect<string, User, Error>(async (userId) => {
    const res = await fetch(`/api/users/${userId}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
});

// Derived stores and events on every effect:
// fetchUserFx.pending  - boolean store, true while running
// fetchUserFx.done     - event fired on success: { params, result }
// fetchUserFx.fail     - event fired on failure: { params, error }
// fetchUserFx.doneData - event fired on success with result only
// fetchUserFx.failData - event fired on failure with error only
// fetchUserFx.finally  - event fired always: { params, status, result?, error? }

const $user = createStore<User | null>(null)
    .on(fetchUserFx.doneData, (_, user) => user)
    .reset(fetchUserFx.fail);

const $isLoading = fetchUserFx.pending;

const $error = createStore<string | null>(null)
    .on(fetchUserFx.failData, (_, err) => err.message)
    .reset(fetchUserFx);
```

Because `pending`, `done`, and `fail` are standard units on every effect, you get loading and error state for free - no boilerplate.

---

### Deriving Stores with `combine`

`combine` creates a derived store whose value is computed from one or more source stores. It updates automatically whenever any source changes.

```typescript
import { combine } from 'effector';

const $firstName = createStore('Jane');
const $lastName = createStore('Smith');

const $fullName = combine($firstName, $lastName, (first, last) => `${first} ${last}`);
// $fullName => "Jane Smith"

// Object syntax - produces a store of { firstName, lastName }
const $formState = combine({ firstName: $firstName, lastName: $lastName });
```

Derived stores are read-only - you can't call `.on` on them directly. Update the source stores and the derived store follows.

---

### `split` - Routing Events to Different Targets

`split` routes a single event to different targets based on a condition:

```typescript
import { split } from 'effector';

const responseReceived = createEvent<{ status: number; body: unknown }>();

const { success, clientError, serverError } = split(responseReceived, {
    success:     ({ status }) => status >= 200 && status < 300,
    clientError: ({ status }) => status >= 400 && status < 500,
    serverError: ({ status }) => status >= 500,
});

// Now each is a separate event you can attach stores or effects to
sample({ clock: serverError, target: notifyOncallFx });
```

---

### Real-World Example: Search with Loading and Error State

Putting it together - a search feature with debounce, loading state, and error handling:

```typescript
import { createStore, createEvent, createEffect, sample, combine } from 'effector';

interface SearchResult {
    id: string;
    title: string;
}

// Events
export const queryChanged = createEvent<string>();
export const searchCleared = createEvent();

// Effects
export const searchFx = createEffect(async (query: string): Promise<SearchResult[]> => {
    const res = await fetch(`/api/products/search?q=${encodeURIComponent(query)}`);
    if (!res.ok) throw new Error('Search failed');
    return res.json();
});

// Stores
export const $query   = createStore('').on(queryChanged, (_, q) => q).reset(searchCleared);
export const $results = createStore<SearchResult[]>([]).on(searchFx.doneData, (_, r) => r).reset(searchCleared);
export const $error   = createStore<string | null>(null).on(searchFx.failData, (_, e) => e.message).reset(queryChanged);

// Derived
export const $isLoading = searchFx.pending;
export const $isEmpty   = combine($results, $isLoading, (results, loading) => !loading && results.length === 0);

// Wire: when query changes and is non-empty, fire search
sample({
    clock: queryChanged,
    source: $query,
    filter: (query) => query.trim().length >= 2,
    target: searchFx,
});
```

The UI layer subscribes to `$results`, `$isLoading`, `$error`, and calls `queryChanged` on input. All logic lives in the model file - the component has nothing to test and nothing to maintain.

---

### Using with React

Effector has a React binding in `effector-react`:

```typescript
import { useUnit } from 'effector-react';
import { $results, $isLoading, $error, $isEmpty, queryChanged } from './model';

export function SearchPage() {
    const [results, isLoading, error, isEmpty] = useUnit([
        $results,
        $isLoading,
        $error,
        $isEmpty,
    ]);
    const handleQuery = useUnit(queryChanged);

    return (
        <div>
            <input
                type="search"
                onChange={(e) => handleQuery(e.target.value)}
                placeholder="Search products..."
            />
            {isLoading && <Spinner />}
            {error && <ErrorBanner message={error} />}
            {isEmpty && <p>No results found.</p>}
            <ul>
                {results.map((r) => <li key={r.id}>{r.title}</li>)}
            </ul>
        </div>
    );
}
```

`useUnit` subscribes to stores and re-renders on change. It also wraps events and effects so they can be called directly from event handlers. The component owns no state - it's a pure view over the model.

---

### Testing

Because Effector logic lives in plain functions with no framework dependency, it tests without any React setup:

```typescript
import { allSettled, fork } from 'effector';
import { $results, $isLoading, $error, queryChanged, searchFx } from './model';

test('successful search populates results', async () => {
    const mockResults = [{ id: '1', title: 'Widget' }];

    const scope = fork({
        handlers: [[searchFx, async () => mockResults]],
    });

    await allSettled(queryChanged, { scope, params: 'widget' });

    expect(scope.getState($results)).toEqual(mockResults);
    expect(scope.getState($error)).toBeNull();
    expect(scope.getState($isLoading)).toBe(false);
});

test('failed search sets error message', async () => {
    const scope = fork({
        handlers: [[searchFx, async () => { throw new Error('Network error'); }]],
    });

    await allSettled(queryChanged, { scope, params: 'widget' });

    expect(scope.getState($error)).toBe('Network error');
    expect(scope.getState($results)).toEqual([]);
});
```

`fork` creates an isolated scope - stores start from their initial values and effects use the provided mock handlers. `allSettled` runs the event and waits for all effects to settle. No mocking frameworks, no async complications.

---

### Effector vs Redux vs Zustand

| | Effector | Redux Toolkit | Zustand |
|---|---|---|---|
| Mental model | Units + explicit data flow | Actions + reducers | Mutable stores + hooks |
| Async handling | Built into effects | Thunks / RTK Query | Manual or middleware |
| Boilerplate | Low (no action types) | Medium | Very low |
| Derived state | `combine`, `sample` | Reselect | Computed via selectors |
| Testing | `fork` / `allSettled` | `configureStore` | Direct calls |
| Bundle size | ~13KB | ~18KB | ~3KB |
| TypeScript | Excellent, inferred | Good | Good |

Effector is the strongest choice when your data flow has multiple sources feeding into shared state - the explicit `sample` wiring makes those relationships visible and testable. Zustand is simpler for small, localised state. Redux Toolkit is worth it if you're already in that ecosystem or need its DevTools.

---

### For AI agents

```
Effector conventions: prefix stores with $, suffix effects with Fx. Use .on(event, reducer) for simple updates. Use sample({ clock, source, filter, fn, target }) for declarative unit wiring. createEffect gives .pending, .doneData, .failData for free. Test with fork({ handlers }) and allSettled() for isolated scopes. Never mutate store state directly.
```

Reference: `https://michalsniezko.github.io/js-frontend-tooling/effector.html`
