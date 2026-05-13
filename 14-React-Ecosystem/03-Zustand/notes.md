# Zustand

## Table of Contents

1. [What is Zustand](#1-what-is-zustand)
2. [Installation](#2-installation)
3. [Creating a Store](#3-creating-a-store)
4. [Accessing State in Components](#4-accessing-state-in-components)
5. [Updating State](#5-updating-state)
6. [Actions Inside Store](#6-actions-inside-store)
7. [Computed Values and Derived State](#7-computed-values-and-derived-state)
8. [Async Actions](#8-async-actions)
9. [Selectors and Re-render Optimization](#9-selectors-and-re-render-optimization)
10. [Slices Pattern for Large Apps](#10-slices-pattern-for-large-apps)
11. [Middleware](#11-middleware)
12. [Persist Middleware](#12-persist-middleware)
13. [Devtools Middleware](#13-devtools-middleware)
14. [Zustand vs Redux Toolkit](#14-zustand-vs-redux-toolkit)
15. [Zustand vs Context API](#15-zustand-vs-context-api)
16. [TypeScript with Zustand](#16-typescript-with-zustand)
17. [Testing Zustand Stores](#17-testing-zustand-stores)
18. [Common Mistakes](#18-common-mistakes)
19. [Best Practices](#19-best-practices)

---

## 1. What is Zustand

Zustand (German for "state") is a small, fast, and scalable state management library for React. It was created by the team behind Jotai and React Spring (Poimandres).

**Core philosophy:**
- Minimal API surface — you only need `create` and a selector function
- No Provider wrapping — the store is a global singleton by default
- No boilerplate — combines state and actions in one object
- Works outside React — can be read and written from plain JavaScript

**Key characteristics:**

| Feature | Detail |
|---|---|
| Bundle size | ~1.5 kB gzipped |
| Provider required | No |
| Immutability | Manual (or Immer middleware) |
| DevTools | Via devtools middleware |
| Async | Native — just use async functions |
| TypeScript | First-class support |

**Mental model:**

```text
create() defines the store
       ↓
Store = { state fields } + { action functions }
       ↓
useStore(selector) subscribes component to selected slice
       ↓
set() triggers re-render only in components whose selected slice changed
```

Zustand does not use React Context internally. It uses a publish-subscribe pattern with `useSyncExternalStore` under the hood, which means updates bypass React's context propagation entirely.

---

## 2. Installation

```bash
npm install zustand
```

No peer dependencies beyond React 18+. No Provider or wrapper component is required anywhere in your component tree.

```jsx
// App.jsx — No <StoreProvider> needed
function App() {
  return (
    <div>
      <Counter />
      <Controls />
    </div>
  );
}
```

This is a major DX advantage over Redux (which requires `<Provider store={store}>`) and Context API (which requires `<Context.Provider value={...}>`).

---

## 3. Creating a Store

The `create` function accepts a callback that receives `set` and `get` and must return an object containing all state fields and all action functions.

```js
import { create } from "zustand";
```

### Basic counter store

```js
// store/useCounterStore.js
import { create } from "zustand";

const useCounterStore = create((set, get) => ({
  // State
  count: 0,
  step: 1,

  // Actions
  increment: () => set((state) => ({ count: state.count + state.step })),
  decrement: () => set((state) => ({ count: state.count - state.step })),
  reset: () => set({ count: 0 }),
  setStep: (newStep) => set({ step: newStep }),

  // Action using get() to read current state synchronously
  doubleIncrement: () => {
    const currentStep = get().step;
    set((state) => ({ count: state.count + currentStep * 2 }));
  },
}));

export default useCounterStore;
```

### The `set` function

`set` performs a **shallow merge** of the object you provide — it does not replace the entire state:

```js
// Only updates count; leaves step and actions unchanged
set({ count: 5 });

// Functional form — safe for derived updates that depend on previous value
set((state) => ({ count: state.count + 1 }));
```

### The `get` function

`get` reads the **current state at the time of the call**, not a stale closure value. It is primarily useful inside actions that need to read other state fields:

```js
const useStore = create((set, get) => ({
  items: [],
  total: 0,
  addItem: (item) => {
    set((state) => ({ items: [...state.items, item] }));
    // Read up-to-date state after set() has been applied
    const updatedItems = get().items;
    const newTotal = updatedItems.reduce((sum, i) => sum + i.price, 0);
    set({ total: newTotal });
  },
}));
```

---

## 4. Accessing State in Components

### With a selector (recommended)

```jsx
function Counter() {
  // Only re-renders when `count` changes
  const count = useCounterStore((state) => state.count);
  return <p>Count: {count}</p>;
}
```

### Accessing actions

Actions are stable function references — they do not change between renders and do not cause re-renders when selected:

```jsx
function Controls() {
  const increment = useCounterStore((state) => state.increment);
  const decrement = useCounterStore((state) => state.decrement);
  const reset = useCounterStore((state) => state.reset);

  return (
    <div>
      <button onClick={decrement}>-</button>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
}
```

### Without a selector (subscribes to entire store)

```jsx
// ❌ Wrong — re-renders on ANY state change in the store
function Counter() {
  const store = useCounterStore();
  return <p>{store.count}</p>;
}

// ✅ Correct — only re-renders when count changes
function Counter() {
  const count = useCounterStore((state) => state.count);
  return <p>{count}</p>;
}
```

---

## 5. Updating State

### Shallow merge (default behavior)

```js
// State: { count: 0, step: 1, name: "counter" }
set({ count: 5 });
// Result: { count: 5, step: 1, name: "counter" }
// ✅ Other fields are preserved automatically
```

### Functional form for derived updates

Always use the functional form when the new value depends on the previous value to avoid stale closures and race conditions:

```js
set((state) => ({ count: state.count + 1 }));
```

### Replacing entire state (replace flag)

```js
// ❌ Dangerous — second arg `true` wipes all state including actions
set({ count: 0 }, true);

// ✅ Full replacement only when intentionally resetting all state
set(initialState, true); // Used in store reset patterns
```

### Nested state updates

Zustand does not deep merge. For nested state, spread manually or use the Immer middleware:

```js
// Manual spread — verbose but explicit
const useProfileStore = create((set) => ({
  user: {
    name: "Alice",
    address: { city: "NYC", zip: "10001" },
  },
  updateCity: (city) =>
    set((state) => ({
      user: {
        ...state.user,
        address: { ...state.user.address, city },
      },
    })),
}));

// With Immer middleware — preferred for deeply nested state
import { immer } from "zustand/middleware/immer";

const useProfileStore = create(
  immer((set) => ({
    user: { name: "Alice", address: { city: "NYC", zip: "10001" } },
    updateCity: (city) =>
      set((state) => {
        state.user.address.city = city; // Direct mutation — safe inside Immer
      }),
  }))
);
```

---

## 6. Actions Inside Store

Defining actions inside `create()` is the idiomatic Zustand pattern. There are no separate action creators, action type constants, or switch-case reducers.

```js
const useCartStore = create((set, get) => ({
  items: [],
  isLoading: false,

  addItem: (product) =>
    set((state) => ({
      items: [...state.items, { ...product, quantity: 1 }],
    })),

  removeItem: (productId) =>
    set((state) => ({
      items: state.items.filter((item) => item.id !== productId),
    })),

  updateQuantity: (productId, quantity) =>
    set((state) => ({
      items: state.items.map((item) =>
        item.id === productId ? { ...item, quantity } : item
      ),
    })),

  // Action using get() — reads current state without closure staleness
  getTotalPrice: () => {
    const { items } = get();
    return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  },

  clearCart: () => set({ items: [] }),
}));
```

**Why this pattern works:**
- Actions are collocated with state — one file per domain
- No dispatch boilerplate — call actions directly
- `get()` prevents stale closures in multi-step actions

---

## 7. Computed Values and Derived State

Zustand does not have a built-in concept of "computed properties" like Pinia or MobX. There are three common patterns:

### Option 1: Compute in selector (recommended)

```jsx
function CartSummary() {
  // Derived value computed in selector — runs on every render, re-renders only when items changes
  const totalPrice = useCartStore(
    (state) =>
      state.items.reduce((sum, item) => sum + item.price * item.quantity, 0)
  );

  return <p>Total: ${totalPrice.toFixed(2)}</p>;
}
```

This is efficient — the component only re-renders when `items` changes and the computed total actually differs (Zustand uses `Object.is` to compare selector results).

### Option 2: Computed getter inside store

```js
const useCartStore = create((set, get) => ({
  items: [],
  // Returns computed value by reading current state via get()
  getTotal: () =>
    get().items.reduce((sum, item) => sum + item.price * item.quantity, 0),
  getItemCount: () => get().items.length,
}));

function CartBadge() {
  const count = useCartStore((state) => state.getItemCount());
  return <span>{count}</span>;
}
```

### Option 3: Memoized selector for expensive computations

```jsx
import { useMemo } from "react";

function ExpensiveList() {
  const items = useCartStore((state) => state.items);
  const sorted = useMemo(
    () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
    [items]
  );
  return (
    <ul>
      {sorted.map((item) => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

---

## 8. Async Actions

Async actions are plain `async` functions inside `create()`. No special middleware, no saga, no thunk.

```js
const useUserStore = create((set) => ({
  user: null,
  isLoading: false,
  error: null,

  fetchUser: async (userId) => {
    set({ isLoading: true, error: null });
    try {
      const response = await fetch(`/api/users/${userId}`);
      if (!response.ok) throw new Error("Failed to fetch user");
      const user = await response.json();
      set({ user, isLoading: false });
    } catch (err) {
      set({ error: err.message, isLoading: false });
    }
  },

  updateUser: async (userId, updates) => {
    set({ isLoading: true });
    try {
      const response = await fetch(`/api/users/${userId}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(updates),
      });
      const updated = await response.json();
      set({ user: updated, isLoading: false });
    } catch (err) {
      set({ error: err.message, isLoading: false });
    }
  },

  clearUser: () => set({ user: null, error: null }),
}));
```

```jsx
function UserProfile({ userId }) {
  const user = useUserStore((state) => state.user);
  const isLoading = useUserStore((state) => state.isLoading);
  const error = useUserStore((state) => state.error);
  const fetchUser = useUserStore((state) => state.fetchUser);

  useEffect(() => {
    fetchUser(userId);
  }, [userId, fetchUser]);

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  if (!user) return null;
  return <h1>{user.name}</h1>;
}
```

**Async state transitions:**

```text
fetchUser(id) called
       ↓
set({ isLoading: true, error: null })   → component shows spinner
       ↓
await fetch(...)
       ↓
  success: set({ user, isLoading: false })   → shows profile
  failure: set({ error: msg, isLoading: false }) → shows error
```

---

## 9. Selectors and Re-render Optimization

### How Zustand tracks subscriptions

Each `useStore(selector)` call registers a subscription. When `set()` is called, Zustand runs every registered selector and compares the new result with the previous result using `Object.is`. If the result changed, the subscribing component re-renders.

### Primitive selector — always safe

```jsx
// Re-renders only when count changes
const count = useStore((state) => state.count);
```

### Object selector problem

```jsx
// ❌ Wrong — new object reference on every render → always triggers re-render
const { count, step } = useStore(
  (state) => ({ count: state.count, step: state.step })
);
```

Even when `count` and `step` have not changed, `{ count, step }` is a new object reference every time the selector runs, so `Object.is` comparison always returns `false`.

### Solution: useShallow

```jsx
import { useShallow } from "zustand/react/shallow";

// ✅ Correct — shallow comparison of each key individually
const { count, step } = useStore(
  useShallow((state) => ({ count: state.count, step: state.step }))
);

// ✅ Also works for array selectors
const [items, total] = useStore(
  useShallow((state) => [state.items, state.total])
);
```

`useShallow` performs a shallow equality check — it compares each key (or index for arrays) in the returned structure using `Object.is`.

### Re-render behavior summary

| Selector type | Comparison | Re-renders when |
|---|---|---|
| Primitive | `Object.is` | Selected value changes |
| Object without `useShallow` | `Object.is` (reference) | Every `set()` call |
| Object with `useShallow` | Shallow equality | Any selected property changes |
| No selector at all | `Object.is` (reference) | Every `set()` call |

---

## 10. Slices Pattern for Large Apps

For large applications, split the store into feature "slices" and combine them into a single unified store.

```js
// store/slices/counterSlice.js
export const createCounterSlice = (set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
});

// store/slices/cartSlice.js
export const createCartSlice = (set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) =>
    set((state) => ({ items: state.items.filter((i) => i.id !== id) })),
});

// store/slices/userSlice.js
export const createUserSlice = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
});

// store/index.js — Combined store
import { create } from "zustand";
import { createCounterSlice } from "./slices/counterSlice";
import { createCartSlice } from "./slices/cartSlice";
import { createUserSlice } from "./slices/userSlice";

const useBoundStore = create((...args) => ({
  ...createCounterSlice(...args),
  ...createCartSlice(...args),
  ...createUserSlice(...args),
}));

export default useBoundStore;
```

### Cross-slice actions

A slice can read state from another slice via `get()` because both live in the same store:

```js
// Inside cartSlice — accesses user from userSlice via get()
export const createCartSlice = (set, get) => ({
  items: [],
  checkout: async () => {
    const { user } = get(); // Reads userSlice state
    if (!user) throw new Error("Must be logged in to checkout");
    await placeOrder(user.id, get().items);
    set({ items: [] });
  },
});
```

---

## 11. Middleware

Middleware wraps the `create` callback to add functionality. Middleware functions are composed by nesting them:

```text
devtools(
  persist(
    immer(
      (set, get, api) => ({ ...state, ...actions })
    )
  )
)
```

```js
import { create } from "zustand";
import { devtools, persist } from "zustand/middleware";
import { immer } from "zustand/middleware/immer";

const useStore = create(
  devtools(
    persist(
      immer((set) => ({
        count: 0,
        increment: () => set((state) => { state.count++; }),
      })),
      { name: "counter-storage" }
    ),
    { name: "CounterStore" }
  )
);
```

### Available official middleware

| Middleware | Import path | Purpose |
|---|---|---|
| `devtools` | `zustand/middleware` | Redux DevTools browser extension |
| `persist` | `zustand/middleware` | localStorage / sessionStorage / custom |
| `immer` | `zustand/middleware/immer` | Write mutations, Immer handles immutability |
| `subscribeWithSelector` | `zustand/middleware` | Subscribe to specific state slices outside React |
| `combine` | `zustand/middleware` | Separate initial state and actions for better inference |

---

## 12. Persist Middleware

`persist` serializes the store to `localStorage` (or any custom storage backend) and rehydrates it on page reload.

### Basic usage

```js
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useAuthStore = create(
  persist(
    (set) => ({
      token: null,
      user: null,
      login: (token, user) => set({ token, user }),
      logout: () => set({ token: null, user: null }),
    }),
    {
      name: "auth-storage", // Key used in localStorage
    }
  )
);
```

### Partial persistence — only save specific keys

```js
persist(
  (set) => ({
    token: null,
    user: null,
    sessionData: null, // Temporary — do not persist
  }),
  {
    name: "auth-storage",
    partialize: (state) => ({
      token: state.token,
      user: state.user,
      // sessionData is excluded — won't be written to storage
    }),
  }
)
```

### Version migration

```js
persist(
  (set) => ({ user: null, preferences: {} }),
  {
    name: "user-storage",
    version: 2, // Increment this when schema changes
    migrate: (persistedState, version) => {
      if (version === 0) {
        // v0 → v1: rename `username` to `user.name`
        persistedState.user = { name: persistedState.username };
        delete persistedState.username;
      }
      if (version === 1) {
        // v1 → v2: add preferences with default value
        persistedState.preferences = { theme: "light" };
      }
      return persistedState;
    },
  }
)
```

### Custom storage backend

```js
persist(
  storeInitializer,
  {
    name: "session-data",
    storage: {
      getItem: (name) => sessionStorage.getItem(name),
      setItem: (name, value) => sessionStorage.setItem(name, value),
      removeItem: (name) => sessionStorage.removeItem(name),
    },
  }
)
```

### Checking rehydration status

```js
// Check synchronously
useStore.persist.hasHydrated();

// Subscribe to hydration lifecycle
useStore.persist.onFinishHydration((state) => {
  console.log("Store rehydrated from storage", state);
});
```

---

## 13. Devtools Middleware

```js
import { create } from "zustand";
import { devtools } from "zustand/middleware";

const useCounterStore = create(
  devtools(
    (set) => ({
      count: 0,
      // Third arg to set() is the action name shown in Redux DevTools
      increment: () =>
        set((state) => ({ count: state.count + 1 }), false, "increment"),
      decrement: () =>
        set((state) => ({ count: state.count - 1 }), false, "decrement"),
      reset: () => set({ count: 0 }, false, "reset"),
    }),
    {
      name: "CounterStore", // Store name in DevTools panel
      enabled: process.env.NODE_ENV !== "production",
    }
  )
);
```

**set() signature with devtools:**

```js
set(updater, replace, actionName)
// updater   — state update (object or function)
// replace   — false (default) = merge; true = replace entire state
// actionName — string label shown in Redux DevTools action log
```

**DevTools features available:**
- State snapshot after every action
- Time-travel debugging — jump to any past state
- Action log with payloads
- State diff view

---

## 14. Zustand vs Redux Toolkit

| Dimension | Zustand | Redux Toolkit |
|---|---|---|
| Bundle size | ~1.5 kB | ~16 kB |
| Boilerplate | Minimal | Moderate |
| Provider required | No | Yes (`<Provider>`) |
| DevTools | Via middleware | Built-in |
| Async handling | Native `async` functions | `createAsyncThunk` |
| Immutability | Manual or Immer middleware | Immer built-in via `createSlice` |
| Learning curve | Low | Moderate |
| TypeScript | Excellent | Excellent |
| RTK Query equivalent | No | Yes (powerful) |
| Middleware ecosystem | Small | Mature |
| Code splitting | Easy (multiple stores) | Possible (slices) |
| Team conventions | Flexible | Structured |

**When to choose Zustand:**
- Small to medium applications
- Teams preferring minimal boilerplate
- No requirement for RTK Query
- Multiple independent stores across features
- Rapid prototyping

**When to choose Redux Toolkit:**
- Large enterprise applications requiring strict conventions
- Complex async data fetching with RTK Query
- Large teams needing enforced patterns
- Existing Redux codebase (migration path)
- Heavy time-travel debugging workflows

---

## 15. Zustand vs Context API

| Dimension | Zustand | Context API |
|---|---|---|
| External dependency | Yes (~1.5 kB) | No |
| Re-render behavior | Only subscribed + changed components | All consumers re-render |
| Performance | Excellent for frequent updates | Degrades with high-frequency updates |
| Setup | No Provider | Requires Provider wrapping |
| Async | Native in store | Manual with useReducer + useEffect |
| DevTools | Via devtools middleware | No built-in support |
| Testing | Easy (reset with setState) | Requires Provider in test wrapper |
| State outside React | Yes (getState, setState, subscribe) | No |

**Context API is sufficient when:**
- State changes infrequently (theme, locale, auth)
- Only a few components consume the value
- No external dependency is preferred

**Zustand is better when:**
- State updates frequently (cart, real-time data, form state)
- Many components subscribe to different slices
- Performance is measurably critical

```text
Context API re-render flow:
Provider value changes → ALL consumers re-render
                        (regardless of which property they read)

Zustand re-render flow:
set() called → each selector re-runs → compare result with Object.is
             → only components with a changed selector result re-render
```

---

## 16. TypeScript with Zustand

### Basic typed store

```ts
import { create } from "zustand";

interface CounterState {
  count: number;
  step: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setStep: (step: number) => void;
}

// Note: create<CounterState>()((set) => ...) — the extra () is required
const useCounterStore = create<CounterState>()((set) => ({
  count: 0,
  step: 1,
  increment: () => set((state) => ({ count: state.count + state.step })),
  decrement: () => set((state) => ({ count: state.count - state.step })),
  reset: () => set({ count: 0 }),
  setStep: (step) => set({ step }),
}));
```

The extra `()` in `create<State>()((set) => ...)` is required to work around TypeScript's inability to infer generic types when using curried functions.

### Typed slices

```ts
import { StateCreator } from "zustand";

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CounterSlice {
  count: number;
  increment: () => void;
}

interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
}

type StoreState = CounterSlice & CartSlice;

const createCounterSlice: StateCreator<StoreState, [], [], CounterSlice> = (
  set
) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
});

const createCartSlice: StateCreator<StoreState, [], [], CartSlice> = (
  set
) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
});

const useBoundStore = create<StoreState>()((...args) => ({
  ...createCounterSlice(...args),
  ...createCartSlice(...args),
}));
```

### Inferring types from store

```ts
// Extract the full state type without defining it separately
type StoreState = ReturnType<typeof useCounterStore.getState>;
```

---

## 17. Testing Zustand Stores

### Testing store actions in isolation

```ts
import { act, renderHook } from "@testing-library/react";
import useCounterStore from "./useCounterStore";

describe("useCounterStore", () => {
  // Reset store to initial state before each test
  beforeEach(() => {
    useCounterStore.setState({ count: 0, step: 1 });
  });

  it("increments count by step", () => {
    const { result } = renderHook(() => useCounterStore((s) => s.count));
    expect(result.current).toBe(0);

    act(() => {
      useCounterStore.getState().increment();
    });

    expect(result.current).toBe(1);
  });

  it("resets count to zero", () => {
    act(() => {
      useCounterStore.setState({ count: 10 });
      useCounterStore.getState().reset();
    });
    expect(useCounterStore.getState().count).toBe(0);
  });
});
```

### Mocking store actions in component tests

```tsx
import { render, screen, fireEvent } from "@testing-library/react";
import Counter from "./Counter";
import useCounterStore from "./useCounterStore";

it("calls increment when button is clicked", () => {
  const increment = jest.fn();
  // Inject mock action into live store
  useCounterStore.setState({ count: 5, increment });

  render(<Counter />);
  fireEvent.click(screen.getByRole("button", { name: /increment/i }));
  expect(increment).toHaveBeenCalledTimes(1);
});
```

### Resetting all stores in Jest setup file

```ts
// jest.setup.ts
import useCounterStore from "./store/useCounterStore";

// Store reference to initial state
const counterInitial = useCounterStore.getInitialState();

afterEach(() => {
  useCounterStore.setState(counterInitial, true);
});
```

### Accessing store outside React

```ts
// Read current state synchronously
const count = useCounterStore.getState().count;

// Write state outside React
useCounterStore.setState({ count: 0 });

// Subscribe to state changes outside React
const unsubscribe = useCounterStore.subscribe(
  (state) => state.count,
  (count, previousCount) => {
    console.log(`count changed: ${previousCount} → ${count}`);
  }
);

unsubscribe(); // Must call this to avoid memory leaks
```

---

## 18. Common Mistakes

### Not using selectors — subscribes to entire store

```jsx
// ❌ Wrong — store is an object, Object.is returns false every render
const store = useCounterStore();
const count = store.count;

// ✅ Correct — selector narrows subscription
const count = useCounterStore((state) => state.count);
```

### Mutating state directly without set()

```js
// ❌ Wrong — bypasses Zustand's subscription system, no re-render
const useStore = create((set, get) => ({
  items: [],
  addItem: (item) => {
    get().items.push(item); // Direct mutation on existing array
  },
}));

// ✅ Correct — return new reference to trigger subscriptions
addItem: (item) =>
  set((state) => ({ items: [...state.items, item] })),
```

### Forgetting useShallow for object selectors

```jsx
// ❌ Wrong — new object every render causes infinite re-renders
const { count, step } = useStore(
  (state) => ({ count: state.count, step: state.step })
);

// ✅ Correct
import { useShallow } from "zustand/react/shallow";
const { count, step } = useStore(
  useShallow((state) => ({ count: state.count, step: state.step }))
);
```

### Creating multiple independent stores for related state

```js
// ❌ Avoid — hard to keep related state in sync across stores
const useCartStore = create(...);
const useInventoryStore = create(...);

// ✅ Better — use one store with slices for related domains
const useAppStore = create((...args) => ({
  ...createCartSlice(...args),
  ...createInventorySlice(...args),
}));
```

### Using the replace flag accidentally

```js
// ❌ Wipes all state and actions from the store
set({ count: 0 }, true);

// ✅ Merge update (default behavior)
set({ count: 0 });
set({ count: 0 }, false); // Explicit false is the same as default
```

---

## 19. Best Practices

### Use selectors to minimize re-renders

Select only the specific fields each component needs. Never subscribe to the whole store unless the component genuinely needs every field.

### Collocate actions with state

Keep actions inside the `create()` callback. Avoid exporting utility functions that call `setState` from outside the store.

### Use slices for multi-domain apps

A single `useBoundStore` with feature slices provides one store (no multiple Provider concerns) while keeping code organized per domain.

### Enable devtools middleware in development only

```js
const useStore = create(
  devtools(
    (set) => ({ ... }),
    { enabled: import.meta.env.DEV }
  )
);
```

### Use Immer for deeply nested state

Avoid spreading 3+ levels deep. Use `zustand/middleware/immer` to write mutations cleanly without error-prone spread chains.

### Always reset store in tests

```ts
beforeEach(() => {
  useStore.setState(initialState, true);
});
```

### Summary reference

| Concern | Recommendation |
|---|---|
| Performance | Always use selectors; use `useShallow` for objects and arrays |
| Complex nested state | Use `immer` middleware |
| Persistence | Use `persist` middleware with `partialize` |
| Debugging | Use `devtools` middleware in development |
| Large apps | Use slices pattern with `StateCreator` type |
| TypeScript | Use `create<State>()()` curried syntax |
| Testing | Reset with `setState(initialState, true)` in `beforeEach` |
| Async | Native `async` functions — no thunk or saga needed |
