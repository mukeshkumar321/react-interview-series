# React useReducer Hook

## Table of Contents

1. [What is useReducer](#1-what-is-usereducer)
2. [The Reducer Function](#2-the-reducer-function)
3. [Dispatching Actions](#3-dispatching-actions)
4. [Complete Example — Counter](#4-complete-example--counter)
5. [Complex Example — Shopping Cart](#5-complex-example--shopping-cart)
6. [useReducer vs useState](#6-usereducer-vs-usestate)
7. [Lazy Initialization](#7-lazy-initialization)
8. [useReducer + useContext Pattern](#8-usereducer--usecontext-pattern)
9. [Why dispatch is Stable](#9-why-dispatch-is-stable)
10. [Immer with useReducer](#10-immer-with-usereducer)
11. [Action Creators](#11-action-creators)
12. [Testing Reducers](#12-testing-reducers)
13. [Common Mistakes](#13-common-mistakes)
14. [Best Practices](#14-best-practices)

---

## 1. What is useReducer

`useReducer` is a React hook for managing state that follows the **reducer pattern** — a well-established pattern from functional programming and popularized in JavaScript by Redux.

When state transitions become complex — multiple sub-values that change together, next state depends on the previous in non-trivial ways, or many different update pathways — `useReducer` gives you a structured, predictable approach.

### Core Syntax

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

| Parameter      | Type       | Description                                                      |
|----------------|------------|------------------------------------------------------------------|
| `reducer`      | Function   | `(state, action) => newState` — pure function                    |
| `initialState` | Any        | The starting value for state                                     |
| `state`        | Any        | Current state value returned from reducer                        |
| `dispatch`     | Function   | Sends an action to the reducer, triggering a re-render if state changes |

### The Three Moving Parts

```text
Component
    │
    │ dispatch({ type: "INCREMENT" })
    ↓
Reducer Function
    │  receives (currentState, action)
    │  computes and returns nextState
    ↓
New State
    │
    ↓
Component re-renders with new state
```

### Mental Model

Think of `useReducer` as a command bus:

- **State** is the current snapshot of your data.
- **Action** is a plain object describing *what happened* (not how to change it).
- **Reducer** is a pure function that takes the current state and an action, and returns the next state.
- **Dispatch** is the function you call to send an action to the reducer.

The reducer is always the single source of truth for how state transitions happen. No scattered `setState` calls — every update flows through one place.

### When to Reach for useReducer

- State object has more than 2–3 sub-values that update in related ways.
- The next state depends on the previous state in complex conditional logic.
- You want to co-locate all state transition logic in one place for readability.
- You are building a feature where multiple actions can affect the same state (forms, carts, wizards).

---

## 2. The Reducer Function

The reducer is the heart of `useReducer`. It is a **pure function** with the signature:

```js
function reducer(state, action) {
  // return the next state
}
```

### Purity Requirements

A reducer **must be pure**:

| Rule                          | Meaning                                                   |
|-------------------------------|-----------------------------------------------------------|
| No side effects               | No API calls, no `console.log`, no DOM manipulation       |
| No mutation of arguments      | Do not modify `state` directly — return a new object      |
| Deterministic                 | Same inputs always produce the same output                |
| No async operations           | Reducers are synchronous; side effects go in `useEffect`  |

React calls your reducer during rendering in `StrictMode` (twice in development) to detect impure reducers. If your reducer has side effects, you will see them fire twice.

### Action Object Convention

The community-standard shape for an action is:

```js
{ type: string, payload?: any }
```

- `type` is a string identifier for the kind of update.
- `payload` carries any data the reducer needs to compute the new state.
- `meta` and `error` are also common in Redux-style patterns but optional.

### Switch Statement Pattern

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 };

    case "DECREMENT":
      return { ...state, count: state.count - 1 };

    case "RESET":
      return { ...state, count: 0 };

    case "SET":
      return { ...state, count: action.payload };

    default:
      return state; // always return state for unrecognized actions
  }
}
```

### ❌ Wrong — Mutating State

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      state.count += 1; // mutates existing state object
      return state;     // returns the same reference — React sees no change
    default:
      return state;
  }
}
```

### ✅ Correct — Returning New State

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + 1 }; // new object reference
    default:
      return state;
  }
}
```

### The Default Case

Always handle the `default` case by returning the current state. This prevents crashes when unknown action types are dispatched (e.g., by a library, a typo, or middleware).

```jsx
default:
  return state;

// or in TypeScript with exhaustive checking:
default:
  throw new Error(`Unhandled action type: ${action.type}`);
```

---

## 3. Dispatching Actions

`dispatch` is the function React gives you to trigger state transitions. You call it with an action object.

### Basic Dispatch

```jsx
dispatch({ type: "INCREMENT" });
dispatch({ type: "DECREMENT" });
dispatch({ type: "RESET" });
```

### Dispatch with Payload

```jsx
dispatch({ type: "SET_NAME", payload: "Alice" });
dispatch({ type: "SET_USER", payload: { id: 1, name: "Alice", role: "admin" } });
dispatch({ type: "SET_COUNT", payload: 42 });
```

### Dispatch Inside Event Handlers

```jsx
function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <div>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "INCREMENT" })}>+</button>
      <button onClick={() => dispatch({ type: "DECREMENT" })}>-</button>
    </div>
  );
}
```

### Dispatch is Stable

The `dispatch` function reference is **guaranteed to be stable** — it does not change between renders. This means:

- You can safely include `dispatch` in `useEffect` and `useCallback` dependency arrays.
- You do not need to memoize it or exclude it from deps.

```jsx
useEffect(() => {
  fetchUser(id).then(user => dispatch({ type: "SET_USER", payload: user }));
}, [id]); // dispatch does NOT need to be in deps, but including it is safe
```

### Batching

React 18 batches all `dispatch` calls made within the same event handler into a single re-render. Calling `dispatch` twice in a synchronous handler does not cause two renders.

```jsx
function handleReset() {
  dispatch({ type: "RESET_ITEMS" });
  dispatch({ type: "RESET_TOTAL" });
  // Only one re-render in React 18
}
```

---

## 4. Complete Example — Counter

### useReducer Version

```jsx
import { useReducer } from "react";

const initialState = { count: 0 };

function counterReducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    case "DECREMENT":
      return { count: state.count - 1 };
    case "RESET":
      return { count: 0 };
    case "SET":
      return { count: action.payload };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "INCREMENT" })}>Increment</button>
      <button onClick={() => dispatch({ type: "DECREMENT" })}>Decrement</button>
      <button onClick={() => dispatch({ type: "RESET" })}>Reset</button>
      <button onClick={() => dispatch({ type: "SET", payload: 10 })}>Set to 10</button>
    </div>
  );
}
```

### Equivalent useState Version

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={() => setCount(c => c - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
      <button onClick={() => setCount(10)}>Set to 10</button>
    </div>
  );
}
```

### Comparison for This Case

For a simple counter, `useState` is perfectly fine. `useReducer` adds boilerplate without benefit. The value of `useReducer` shows up when the state shape grows and multiple fields interact.

---

## 5. Complex Example — Shopping Cart

This example demonstrates where `useReducer` shines — a shopping cart where multiple fields of state must update atomically.

### State Shape

```js
const initialState = {
  items: [],      // Array of { id, name, price, quantity }
  total: 0,       // Computed sum
  discount: 0,    // Applied discount percentage
  isLoading: false,
};
```

### Action Types

```js
const ACTIONS = {
  ADD_ITEM:       "ADD_ITEM",
  REMOVE_ITEM:    "REMOVE_ITEM",
  UPDATE_QTY:     "UPDATE_QTY",
  APPLY_DISCOUNT: "APPLY_DISCOUNT",
  CLEAR_CART:     "CLEAR_CART",
  SET_LOADING:    "SET_LOADING",
};
```

### The Reducer

```jsx
function cartReducer(state, action) {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find(i => i.id === action.payload.id);
      const updatedItems = existing
        ? state.items.map(i =>
            i.id === action.payload.id
              ? { ...i, quantity: i.quantity + 1 }
              : i
          )
        : [...state.items, { ...action.payload, quantity: 1 }];

      const rawTotal = updatedItems.reduce(
        (sum, i) => sum + i.price * i.quantity,
        0
      );
      const total = rawTotal * (1 - state.discount / 100);

      return { ...state, items: updatedItems, total };
    }

    case "REMOVE_ITEM": {
      const updatedItems = state.items.filter(i => i.id !== action.payload);
      const rawTotal = updatedItems.reduce(
        (sum, i) => sum + i.price * i.quantity,
        0
      );
      const total = rawTotal * (1 - state.discount / 100);
      return { ...state, items: updatedItems, total };
    }

    case "UPDATE_QTY": {
      const { id, quantity } = action.payload;
      const updatedItems = state.items.map(i =>
        i.id === id ? { ...i, quantity } : i
      );
      const rawTotal = updatedItems.reduce(
        (sum, i) => sum + i.price * i.quantity,
        0
      );
      const total = rawTotal * (1 - state.discount / 100);
      return { ...state, items: updatedItems, total };
    }

    case "APPLY_DISCOUNT": {
      const discount = action.payload;
      const rawTotal = state.items.reduce(
        (sum, i) => sum + i.price * i.quantity,
        0
      );
      const total = rawTotal * (1 - discount / 100);
      return { ...state, discount, total };
    }

    case "CLEAR_CART":
      return { ...initialState };

    case "SET_LOADING":
      return { ...state, isLoading: action.payload };

    default:
      return state;
  }
}
```

### Usage

```jsx
function ShoppingCart() {
  const [cart, dispatch] = useReducer(cartReducer, initialState);

  const addItem = (product) =>
    dispatch({ type: "ADD_ITEM", payload: product });

  const removeItem = (id) =>
    dispatch({ type: "REMOVE_ITEM", payload: id });

  const applyDiscount = (pct) =>
    dispatch({ type: "APPLY_DISCOUNT", payload: pct });

  return (
    <div>
      {cart.items.map(item => (
        <div key={item.id}>
          {item.name} x{item.quantity}
          <button onClick={() => removeItem(item.id)}>Remove</button>
        </div>
      ))}
      <p>Total: ${cart.total.toFixed(2)}</p>
      <button onClick={() => applyDiscount(10)}>Apply 10% Discount</button>
      <button onClick={() => dispatch({ type: "CLEAR_CART" })}>Clear</button>
    </div>
  );
}
```

Observe that every action that changes items also recalculates `total` atomically. With `useState` this would require multiple setter calls or careful effect coordination.

---

## 6. useReducer vs useState

### Decision Table

| Criterion                          | useState                    | useReducer                           |
|------------------------------------|-----------------------------|--------------------------------------|
| State shape                        | Scalar or flat object       | Complex / nested object              |
| Number of fields                   | 1–3                         | 4+                                   |
| State transitions                  | Simple assignments          | Conditional, multi-field updates     |
| Next state depends on previous     | Functional updater handles it | Natural fit                         |
| Multiple related fields update     | Requires multiple setters   | Single dispatch, one atomic update   |
| Transition logic location          | Scattered in handlers       | Centralized in reducer               |
| Testing state logic                | Tied to component           | Pure function — easily unit-tested   |
| Action history / debugging         | Manual                      | DevTools-compatible pattern          |

### When useState Wins

```jsx
// Independent, simple values — useState is cleaner
const [isOpen, setIsOpen] = useState(false);
const [name, setName] = useState("");
const [count, setCount] = useState(0);
```

### When useReducer Wins

```jsx
// Related values with complex transitions — useReducer is cleaner
const [form, dispatch] = useReducer(formReducer, {
  username: "",
  email: "",
  password: "",
  errors: {},
  isSubmitting: false,
  isSuccess: false,
});
```

### Complexity Threshold Rule of Thumb

```text
If you find yourself writing more than 3 related useState calls
that update together in the same event handler → reach for useReducer.
```

---

## 7. Lazy Initialization

`useReducer` accepts an optional **third argument** — an `init` function. React calls `init(initialArg)` once on mount to compute the actual initial state. This is **lazy initialization**.

### Syntax

```jsx
const [state, dispatch] = useReducer(reducer, initialArg, init);
```

- `initialArg` is passed as the argument to `init`.
- `init(initialArg)` returns the actual initial state.
- The `init` function runs **only once** — on the initial render.

### Use Case 1 — Expensive Computation

```jsx
function buildInitialState(userId) {
  // Imagine this reads from a large dataset or performs calculation
  const savedData = localStorage.getItem(`cart_${userId}`);
  return savedData ? JSON.parse(savedData) : { items: [], total: 0 };
}

function CartPage({ userId }) {
  const [state, dispatch] = useReducer(cartReducer, userId, buildInitialState);
  // buildInitialState(userId) called once on mount
}
```

### ❌ Wrong — Expensive computation runs every render

```jsx
function CartPage({ userId }) {
  // This recalculates on every render (wasteful)
  const initial = buildInitialState(userId);
  const [state, dispatch] = useReducer(cartReducer, initial);
}
```

### ✅ Correct — Lazy init runs only once

```jsx
function CartPage({ userId }) {
  const [state, dispatch] = useReducer(cartReducer, userId, buildInitialState);
}
```

### Use Case 2 — Reset to Initial State

Lazy init also enables a clean reset pattern — dispatch the `initialArg` as part of a RESET action and call `init` inside the reducer:

```jsx
function reducer(state, action) {
  switch (action.type) {
    case "RESET":
      return init(action.payload); // re-use the init function
    // ...
    default:
      return state;
  }
}
```

---

## 8. useReducer + useContext Pattern

This pattern combines `useReducer` for state management with `useContext` for distributing that state and dispatch through the component tree — a lightweight alternative to Redux for medium-complexity apps.

### Flow Diagram

```text
Provider Component
  │  holds: const [state, dispatch] = useReducer(...)
  │
  ├─ StateContext.Provider value={state}
  └─ DispatchContext.Provider value={dispatch}
          │
          ├─ ChildA  → reads state via useContext(StateContext)
          ├─ ChildB  → calls dispatch via useContext(DispatchContext)
          └─ ChildC  → both
```

### Implementation

```jsx
import { createContext, useContext, useReducer } from "react";

// 1. Create separate contexts for state and dispatch
const StateContext = createContext(null);
const DispatchContext = createContext(null);

// 2. Reducer
function appReducer(state, action) {
  switch (action.type) {
    case "SET_USER":
      return { ...state, user: action.payload };
    case "TOGGLE_THEME":
      return { ...state, theme: state.theme === "light" ? "dark" : "light" };
    case "SET_LOADING":
      return { ...state, isLoading: action.payload };
    default:
      return state;
  }
}

const initialState = {
  user: null,
  theme: "light",
  isLoading: false,
};

// 3. Provider component
export function AppProvider({ children }) {
  const [state, dispatch] = useReducer(appReducer, initialState);

  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  );
}

// 4. Custom hooks for consuming
export function useAppState() {
  const context = useContext(StateContext);
  if (!context) throw new Error("useAppState must be used within AppProvider");
  return context;
}

export function useAppDispatch() {
  const context = useContext(DispatchContext);
  if (!context) throw new Error("useAppDispatch must be used within AppProvider");
  return context;
}
```

### Consumer Components

```jsx
// Reads state
function Header() {
  const { user, theme } = useAppState();
  return <header className={theme}>{user?.name}</header>;
}

// Dispatches actions
function ThemeToggle() {
  const dispatch = useAppDispatch();
  return (
    <button onClick={() => dispatch({ type: "TOGGLE_THEME" })}>
      Toggle Theme
    </button>
  );
}
```

### Why Separate State and Dispatch Contexts?

If you put both in a single context, components that only dispatch (never read state) will still re-render whenever state changes. Separating them means dispatch-only consumers never re-render due to state changes.

---

## 9. Why dispatch is Stable

React guarantees that the `dispatch` function returned by `useReducer` has a **stable identity** — it is the same function reference across all renders of the component.

### What This Means

```jsx
const [state, dispatch] = useReducer(reducer, initialState);

// On render 1: dispatch === fn_A
// On render 2: dispatch === fn_A  (same reference)
// On render 100: dispatch === fn_A  (same reference)
```

### Practical Implication 1 — useEffect Dependencies

```jsx
useEffect(() => {
  const socket = connectSocket();
  socket.on("message", (msg) => {
    dispatch({ type: "ADD_MESSAGE", payload: msg });
  });
  return () => socket.disconnect();
}, []); // dispatch can be omitted — but ESLint's exhaustive-deps won't warn
        // because React guarantees its stability
```

### Practical Implication 2 — useCallback Dependencies

```jsx
const handleSubmit = useCallback(() => {
  dispatch({ type: "SUBMIT" });
}, []); // stable — dispatch never changes
```

### Note on useState Setters

`setState` from `useState` is also stable — the setter function reference never changes either. The stability guarantee applies to both hooks.

---

## 10. Immer with useReducer

Deep nested state updates using pure spread syntax become verbose and error-prone:

### ❌ Without Immer — Verbose Nested Spread

```jsx
case "UPDATE_ADDRESS_CITY":
  return {
    ...state,
    user: {
      ...state.user,
      address: {
        ...state.user.address,
        city: action.payload,
      },
    },
  };
```

### ✅ With Immer — Mutable-Style Syntax (Still Immutable Under the Hood)

```bash
npm install immer
```

```jsx
import { produce } from "immer";

function reducer(state, action) {
  return produce(state, draft => {
    switch (action.type) {
      case "UPDATE_ADDRESS_CITY":
        draft.user.address.city = action.payload;
        break;

      case "ADD_ITEM":
        draft.cart.items.push(action.payload);
        draft.cart.total += action.payload.price;
        break;

      case "REMOVE_ITEM":
        draft.cart.items = draft.cart.items.filter(
          i => i.id !== action.payload
        );
        break;
    }
  });
}
```

`produce` creates a structural clone, lets you mutate the `draft`, and then freezes and returns the new immutable state. React sees a new reference and re-renders.

### useImmerReducer (from use-immer library)

```bash
npm install use-immer
```

```jsx
import { useImmerReducer } from "use-immer";

function reducer(draft, action) {
  // draft is already a mutable proxy — no need to call produce
  switch (action.type) {
    case "UPDATE_CITY":
      draft.user.address.city = action.payload;
      break;
  }
}

function Profile() {
  const [state, dispatch] = useImmerReducer(reducer, initialState);
}
```

---

## 11. Action Creators

Action creators are functions that return action objects. They improve readability, reduce typos, and centralize the shape of actions.

### Without Action Creators

```jsx
dispatch({ type: "SET_USER", payload: { id: 1, name: "Alice" } });
dispatch({ type: "SET_LOADING", payload: true });
dispatch({ type: "ADD_ITEM", payload: { id: 5, name: "Widget", price: 9.99 } });
```

### With Action Creators

```jsx
// Action creators
const setUser = (user) => ({ type: "SET_USER", payload: user });
const setLoading = (bool) => ({ type: "SET_LOADING", payload: bool });
const addItem = (item) => ({ type: "ADD_ITEM", payload: item });

// Usage
dispatch(setUser({ id: 1, name: "Alice" }));
dispatch(setLoading(true));
dispatch(addItem({ id: 5, name: "Widget", price: 9.99 }));
```

### TypeScript — Typed Actions

```ts
type Action =
  | { type: "INCREMENT" }
  | { type: "DECREMENT" }
  | { type: "SET"; payload: number }
  | { type: "RESET" };

function reducer(state: { count: number }, action: Action): { count: number } {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    case "DECREMENT":
      return { count: state.count - 1 };
    case "SET":
      return { count: action.payload }; // TypeScript knows payload is number
    case "RESET":
      return { count: 0 };
  }
}
```

---

## 12. Testing Reducers

Because reducers are pure functions, they are the most testable unit in a React application. You do not need React, a DOM, or a test renderer to test them.

### Unit Testing a Reducer

```js
// counterReducer.test.js
import { counterReducer } from "./counterReducer";

describe("counterReducer", () => {
  const initial = { count: 0 };

  test("INCREMENT increases count by 1", () => {
    const next = counterReducer(initial, { type: "INCREMENT" });
    expect(next.count).toBe(1);
  });

  test("DECREMENT decreases count by 1", () => {
    const state = { count: 5 };
    const next = counterReducer(state, { type: "DECREMENT" });
    expect(next.count).toBe(4);
  });

  test("RESET returns count to 0", () => {
    const state = { count: 99 };
    const next = counterReducer(state, { type: "RESET" });
    expect(next.count).toBe(0);
  });

  test("SET sets count to payload", () => {
    const next = counterReducer(initial, { type: "SET", payload: 42 });
    expect(next.count).toBe(42);
  });

  test("unknown action returns current state unchanged", () => {
    const next = counterReducer(initial, { type: "UNKNOWN" });
    expect(next).toBe(initial); // same reference
  });

  test("does not mutate previous state", () => {
    const state = { count: 10 };
    const frozen = Object.freeze(state); // will throw if mutated
    expect(() =>
      counterReducer(frozen, { type: "INCREMENT" })
    ).not.toThrow();
  });
});
```

### Testing Components with useReducer

```jsx
import { render, screen, fireEvent } from "@testing-library/react";
import Counter from "./Counter";

test("increments count on button click", () => {
  render(<Counter />);
  fireEvent.click(screen.getByText("Increment"));
  expect(screen.getByText(/Count: 1/)).toBeInTheDocument();
});
```

---

## 13. Common Mistakes

### Mistake 1 — Mutating State in the Reducer

```jsx
// ❌ Mutation — React uses Object.is comparison
case "ADD_TAG":
  state.tags.push(action.payload); // mutates in place
  return state; // same reference → React skips re-render

// ✅ New reference
case "ADD_TAG":
  return { ...state, tags: [...state.tags, action.payload] };
```

### Mistake 2 — Not Handling the Default Case

```jsx
// ❌ Missing default — returns undefined for unknown actions
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    // no default → crashes when unknown action dispatched
  }
}

// ✅ Always return current state in default
function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { count: state.count + 1 };
    default:
      return state;
  }
}
```

### Mistake 3 — Side Effects Inside the Reducer

```jsx
// ❌ API call inside reducer — impure
case "FETCH_USER":
  fetch(`/api/user/${action.payload}`) // side effect in reducer
    .then(r => r.json())
    .then(user => /* can't do anything here */);
  return state;

// ✅ Dispatch to set loading, fetch in useEffect or event handler
function fetchAndSetUser(id, dispatch) {
  dispatch({ type: "SET_LOADING", payload: true });
  fetch(`/api/user/${id}`)
    .then(r => r.json())
    .then(user => dispatch({ type: "SET_USER", payload: user }))
    .finally(() => dispatch({ type: "SET_LOADING", payload: false }));
}
```

### Mistake 4 — Re-creating initialState Object on Every Render

```jsx
// ❌ New object reference every render (minor issue, but wasteful)
function MyComponent() {
  const [state, dispatch] = useReducer(reducer, { count: 0, name: "" });
}

// ✅ Define outside the component (stable reference)
const initialState = { count: 0, name: "" };

function MyComponent() {
  const [state, dispatch] = useReducer(reducer, initialState);
}
```

### Mistake 5 — Dispatching Actions from Inside the Reducer

```jsx
// ❌ Calling dispatch inside reducer — illegal
function reducer(state, action) {
  case "DO_BOTH":
    dispatch({ type: "OTHER" }); // ERROR: dispatch is not in scope here
    return { ...state };
}

// ✅ Dispatch both actions from the component handler
function handleBoth() {
  dispatch({ type: "ACTION_ONE" });
  dispatch({ type: "ACTION_TWO" });
}
```

---

## 14. Best Practices

### Use useReducer for Forms with Many Fields

```jsx
// Managing form state — useReducer is a natural fit
function formReducer(state, action) {
  switch (action.type) {
    case "SET_FIELD":
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        errors: { ...state.errors, [action.field]: "" }, // clear error on change
      };
    case "SET_ERROR":
      return {
        ...state,
        errors: { ...state.errors, [action.field]: action.error },
      };
    case "SUBMIT_START":
      return { ...state, isSubmitting: true };
    case "SUBMIT_SUCCESS":
      return { ...state, isSubmitting: false, isSuccess: true };
    case "SUBMIT_ERROR":
      return { ...state, isSubmitting: false, submitError: action.error };
    default:
      return state;
  }
}
```

### Co-locate Reducer with Component File for Small Cases

For small components, define the reducer in the same file. Only move it to a separate file when it grows large enough to justify it.

### Use String Constants for Action Types

```jsx
// actions.js
export const ACTIONS = {
  INCREMENT:  "INCREMENT",
  DECREMENT:  "DECREMENT",
  RESET:      "RESET",
};

// counterReducer.js
import { ACTIONS } from "./actions";

function reducer(state, action) {
  switch (action.type) {
    case ACTIONS.INCREMENT: return { count: state.count + 1 };
    case ACTIONS.DECREMENT: return { count: state.count - 1 };
    default: return state;
  }
}
```

### Keep State Serializable

Avoid putting functions, class instances, or DOM nodes in the reducer state. State should be plain JSON-serializable data. This keeps state predictable, debuggable, and compatible with persistence or DevTools.

### Summary Table — When to Use Each Pattern

| State Complexity    | Pattern                          |
|---------------------|----------------------------------|
| Single boolean      | `useState`                       |
| 2–3 independent vals| `useState` (multiple)            |
| Complex object      | `useReducer`                     |
| Global + complex    | `useReducer` + `useContext`      |
| App-wide + large    | Redux Toolkit / Zustand / Jotai  |

---
