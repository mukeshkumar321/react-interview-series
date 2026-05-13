## 📚 Zustand — Tricky Output Questions

> These questions test deep understanding of Zustand's subscription model, selector optimization, action behavior, async state transitions, and persistence. Each question is designed to reflect real interview scenarios.

---

## 1. Basic Store

### Q1

```jsx
import { create } from "zustand";

const useStore = create((set) => ({
  count: 0,
  name: "Alice",
}));

function App() {
  const count = useStore((state) => state.count);
  console.log("render", count);
  return <p>{count}</p>;
}
```

The component mounts. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
render 0
```

**Explanation:** On initial mount, the selector runs once and returns `0`. The component renders once and logs `"render 0"`. No actions have been called yet so there are no subsequent renders.

</details>

---

### Q2

```jsx
const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

function App() {
  const count = useStore((state) => state.count);
  const increment = useStore((state) => state.increment);

  console.log("render");

  return <button onClick={increment}>Click</button>;
}
```

The button is clicked 3 times. How many times does "render" log in total?

<details>
<summary>✅ Answer</summary>

```txt
render       ← initial mount
render       ← after click 1 (count: 0 → 1)
render       ← after click 2 (count: 1 → 2)
render       ← after click 3 (count: 2 → 3)
```

Total: 4 times.

**Explanation:** The component subscribes to `count` via a selector. Each `increment` call changes `count`, so the selector result changes and the component re-renders. The `increment` function itself is a stable reference and does not cause re-renders when selected on its own.

</details>

---

### Q3

```jsx
const useStore = create((set) => ({
  count: 0,
  text: "hello",
  increment: () => set((state) => ({ count: state.count + 1 })),
  setText: (t) => set({ text: t }),
}));

function CountDisplay() {
  const count = useStore((state) => state.count);
  console.log("CountDisplay render");
  return <p>{count}</p>;
}

function TextDisplay() {
  const text = useStore((state) => state.text);
  console.log("TextDisplay render");
  return <p>{text}</p>;
}
```

`useStore.getState().setText("world")` is called once. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
TextDisplay render
```

**Explanation:** `CountDisplay` subscribes only to `count`. `TextDisplay` subscribes only to `text`. When `setText("world")` is called, only the `text` selector result changes. `CountDisplay`'s selector still returns `0`, so `Object.is(0, 0)` is `true` — no re-render. Only `TextDisplay` re-renders. This is the core advantage of Zustand's subscription model.

</details>

---

### Q4

```js
const useStore = create((set) => ({
  a: 1,
  b: 2,
}));

useStore.setState({ a: 10 });
console.log(useStore.getState());
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
{ a: 10, b: 2 }
```

**Explanation:** `setState` (and `set` inside actions) performs a **shallow merge**. Only `a` is updated; `b` is preserved. The entire state is not replaced unless the second argument `true` is passed.

</details>

---

## 2. Selectors

### Q5

```jsx
const useStore = create((set) => ({
  user: { name: "Alice", age: 30 },
  score: 100,
  updateScore: () => set((state) => ({ score: state.score + 1 })),
}));

function UserCard() {
  const user = useStore((state) => state.user);
  console.log("UserCard render");
  return <p>{user.name}</p>;
}
```

`useStore.getState().updateScore()` is called. Does `UserCard` re-render?

<details>
<summary>✅ Answer</summary>

```txt
No — UserCard does NOT re-render.
```

**Explanation:** `UserCard`'s selector returns `state.user`. `updateScore()` only modifies `score`. The `user` object reference has not changed, so `Object.is(prevUser, nextUser)` returns `true`. Zustand skips the re-render. This demonstrates that selector-based subscriptions are precise at the reference level.

</details>

---

### Q6

```jsx
import { useShallow } from "zustand/react/shallow";

const useStore = create((set) => ({
  count: 0,
  step: 1,
  increment: () => set((state) => ({ count: state.count + state.step })),
}));

function Controls() {
  // Version A
  const { count, step } = useStore((state) => ({
    count: state.count,
    step: state.step,
  }));

  // Version B
  const { count, step } = useStore(
    useShallow((state) => ({ count: state.count, step: state.step }))
  );

  console.log("render");
}
```

Assume only one version runs. An unrelated action `useStore.setState({ unrelated: true })` is called. Which version re-renders?

<details>
<summary>✅ Answer</summary>

```txt
Version A: re-renders (❌ incorrect behavior)
Version B: does NOT re-render (✅ correct behavior)
```

**Explanation:** Version A creates a new object `{ count, step }` on every selector invocation. Even though `count` and `step` values are unchanged, `Object.is(oldObj, newObj)` is `false` because they are different object references. Version B uses `useShallow`, which compares `count` and `step` values individually. Since neither changed, the shallow comparison returns `true` and no re-render occurs.

</details>

---

### Q7

```jsx
const useStore = create((set) => ({
  items: ["a", "b", "c"],
  addItem: (item) =>
    set((state) => ({ items: [...state.items, item] })),
}));

function ItemList() {
  const items = useStore((state) => state.items);
  console.log("render", items.length);
  return null;
}
```

`addItem("d")` is called. What is logged during this event?

<details>
<summary>✅ Answer</summary>

```txt
render 4
```

**Explanation:** `addItem("d")` creates a new array `[...state.items, "d"]`. The selector returns `state.items`. The new items array is a different reference, so `Object.is(prev, next)` is `false`. The component re-renders and logs `"render 4"`.

</details>

---

### Q8

```jsx
const useStore = create((set) => ({
  items: [],
  count: 0,
}));

function Component() {
  const everything = useStore(); // No selector
  console.log("render");
  return null;
}
```

`useStore.setState({ count: 1 })` is called. Then `useStore.setState({ count: 1 })` is called again with the same value. How many times is "render" logged after the initial mount?

<details>
<summary>✅ Answer</summary>

```txt
render  ← initial mount
render  ← after first setState({ count: 1 })
render  ← after second setState({ count: 1 })
```

Total after mount: 2 additional renders.

**Explanation:** Without a selector, the component subscribes to the entire store object. Every `set()` call produces a new store state object reference. `Object.is(prevStore, nextStore)` is always `false` for different calls, even when the values are identical. This is why using selectors is critical.

</details>

---

## 3. Actions

### Q9

```js
const useStore = create((set) => ({
  count: 0,
  step: 5,
  incrementByStep: () =>
    set((state) => ({ count: state.count + state.step })),
}));

useStore.getState().incrementByStep();
useStore.getState().incrementByStep();
console.log(useStore.getState().count);
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
10
```

**Explanation:** Each call to `incrementByStep` adds `step` (which is `5`) to `count`. After two calls: `0 + 5 + 5 = 10`. The functional form `set((state) => ...)` reads the latest state on each call, so there is no stale closure issue.

</details>

---

### Q10

```js
const useStore = create((set, get) => ({
  count: 0,
  multiplier: 3,
  multiplyCount: () => {
    const { count, multiplier } = get();
    set({ count: count * multiplier });
  },
}));

useStore.setState({ count: 4 });
useStore.getState().multiplyCount();
console.log(useStore.getState().count);
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
12
```

**Explanation:** `setState({ count: 4 })` sets count to 4. `multiplyCount` calls `get()` to read the current state (`count: 4, multiplier: 3`) and calls `set({ count: 4 * 3 })`. Result: `12`. `get()` is the key — it reads the live state at call time, not a stale closure.

</details>

---

### Q11

```js
const useStore = create((set, get) => ({
  items: [],
  isLoading: false,
  fetchItems: async () => {
    set({ isLoading: true });
    await new Promise((r) => setTimeout(r, 0));
    set({ items: ["x", "y"], isLoading: false });
  },
}));

useStore.getState().fetchItems();
console.log(useStore.getState().isLoading); // Line A
```

What does Line A log? What is the state after the async operation completes?

<details>
<summary>✅ Answer</summary>

```txt
Line A: true
After async completes: { isLoading: false, items: ["x", "y"] }
```

**Explanation:** `fetchItems` is `async`. After the first `set({ isLoading: true })`, the function suspends at `await`. Line A runs synchronously and reads `isLoading: true`. After the microtask/timeout resolves, the second `set` runs and updates state to `{ items: ["x", "y"], isLoading: false }`.

</details>

---

### Q12

```js
const useStore = create((set) => ({
  count: 0,
  batchUpdate: () => {
    set({ count: 1 });
    set({ count: 2 });
    set({ count: 3 });
  },
}));

useStore.getState().batchUpdate();
console.log(useStore.getState().count);
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
3
```

**Explanation:** All three `set()` calls happen synchronously. In React 18, state updates inside event handlers and synchronous code are batched, but Zustand processes each `set()` immediately — the store state is updated synchronously in order. After all three calls, `count` is `3`. However, React will only trigger one re-render for the batch.

</details>

---

## 4. Computed and Derived State

### Q13

```jsx
const useStore = create((set) => ({
  prices: [10, 20, 30],
}));

function Total() {
  const total = useStore(
    (state) => state.prices.reduce((sum, p) => sum + p, 0)
  );
  console.log("render", total);
  return <p>{total}</p>;
}
```

`useStore.setState({ prices: [10, 20, 30] })` is called (same values, new array). Does `Total` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — Total re-renders.
Logs: render 60
```

**Explanation:** The selector computes `60` from both the old and new arrays. However, Zustand compares selector results using `Object.is`. The old result is `60` (a number) and the new result is `60` (a number). `Object.is(60, 60)` is `true`. Therefore, `Total` does **not** re-render. Wait — but `setState` with a new array changes the `prices` reference. The selector runs and computes `60`. Previous selector result was also `60`. `Object.is(60, 60) === true`. No re-render.

**Correction:** Total does **NOT** re-render because `Object.is(60, 60)` is `true`. The selector returns a primitive and Zustand's comparison correctly detects no change.

</details>

---

### Q14

```js
const useStore = create((set, get) => ({
  base: 10,
  multiplier: 2,
  getResult: () => get().base * get().multiplier,
}));

useStore.setState({ multiplier: 5 });
console.log(useStore.getState().getResult());
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
50
```

**Explanation:** `getResult` uses `get()` which always reads the current state. After `setState({ multiplier: 5 })`, `get().base` is `10` and `get().multiplier` is `5`. Result: `10 * 5 = 50`.

</details>

---

### Q15

```jsx
const useStore = create((set) => ({
  items: [{ id: 1, name: "A" }, { id: 2, name: "B" }],
}));

function ItemCount() {
  const count = useStore((state) => state.items.length);
  console.log("render", count);
  return <p>{count}</p>;
}
```

`useStore.setState({ items: [{ id: 1, name: "A" }, { id: 2, name: "B" }, { id: 3, name: "C" }] })` is called. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
render 3
```

**Explanation:** The selector returns `state.items.length`. Before: `2`. After: `3`. `Object.is(2, 3)` is `false`. The component re-renders and logs `"render 3"`.

</details>

---

## 5. Persist Middleware

### Q16

```js
import { create } from "zustand";
import { persist } from "zustand/middleware";

const useStore = create(
  persist(
    (set) => ({
      token: null,
      login: (t) => set({ token: t }),
    }),
    { name: "auth" }
  )
);

useStore.getState().login("abc123");
// Simulate page reload — store re-initializes from localStorage
const rehydrated = useStore.getState().token;
console.log(rehydrated);
```

What is logged (assuming the persist middleware has rehydrated successfully)?

<details>
<summary>✅ Answer</summary>

```txt
"abc123"
```

**Explanation:** `persist` middleware serializes state to `localStorage` under the key `"auth"` on every `set()` call. On store initialization (simulating a page reload), it reads from `localStorage` and calls `set()` to restore state. `token` is rehydrated to `"abc123"`.

</details>

---

### Q17

```js
const useStore = create(
  persist(
    (set) => ({
      token: null,
      tempData: "session-only",
    }),
    {
      name: "app-storage",
      partialize: (state) => ({ token: state.token }),
    }
  )
);

useStore.getState(); // After rehydration from storage: { token: "saved-token" }
console.log(useStore.getState().tempData);
```

Assuming localStorage contained `{ token: "saved-token" }`, what does `tempData` equal after rehydration?

<details>
<summary>✅ Answer</summary>

```txt
"session-only"
```

**Explanation:** `partialize` limits what gets written to storage — only `token` is persisted. `tempData` is never written to `localStorage`. On rehydration, Zustand merges the persisted `{ token: "saved-token" }` with the `initialState`. Since `tempData` is not in storage, it falls back to its initial value `"session-only"`.

</details>

---

### Q18

```js
const useStore = create(
  persist(
    (set) => ({ count: 0 }),
    {
      name: "counter",
      version: 1,
      migrate: (state, version) => {
        if (version === 0) {
          return { count: state.value ?? 0 }; // v0 used `value` key
        }
        return state;
      },
    }
  )
);
```

Stored in localStorage: `{ "state": { "value": 42 }, "version": 0 }`. What is `useStore.getState().count` after rehydration?

<details>
<summary>✅ Answer</summary>

```txt
42
```

**Explanation:** The stored version is `0`, but the current version is `1`. Since they differ, Zustand calls `migrate(persistedState, 0)`. The migrate function maps `value` → `count`, returning `{ count: 42 }`. This migrated state is applied to the store.

</details>

---

## 6. Zustand vs Context

### Q19

```jsx
const CountContext = React.createContext(0);

function Parent() {
  const [state, setState] = useState({ count: 0, name: "Alice" });
  return (
    <CountContext.Provider value={state}>
      <ChildA />
      <ChildB />
    </CountContext.Provider>
  );
}

function ChildA() {
  const { count } = useContext(CountContext);
  console.log("ChildA render");
  return <p>{count}</p>;
}

function ChildB() {
  const { name } = useContext(CountContext);
  console.log("ChildB render");
  return <p>{name}</p>;
}
```

`setState({ count: 1, name: "Alice" })` is called (count changes, name stays the same). What is logged?

<details>
<summary>✅ Answer</summary>

```txt
ChildA render
ChildB render
```

**Explanation:** React Context has no fine-grained subscription. Both `ChildA` and `ChildB` are consumers of `CountContext`. When the Provider's `value` changes (new object reference), all consumers re-render, regardless of which properties they actually use. This is the fundamental performance limitation of Context API for frequently changing state.

</details>

---

### Q20

```jsx
const useStore = create((set) => ({
  count: 0,
  name: "Alice",
  setCount: (n) => set({ count: n }),
}));

function ChildA() {
  const count = useStore((state) => state.count);
  console.log("ChildA render");
  return <p>{count}</p>;
}

function ChildB() {
  const name = useStore((state) => state.name);
  console.log("ChildB render");
  return <p>{name}</p>;
}
```

`useStore.getState().setCount(1)` is called. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
ChildA render
```

**Explanation:** `setCount(1)` updates only `count`. `ChildA` subscribes to `count` — it changes from `0` to `1`, so it re-renders. `ChildB` subscribes to `name` — it remains `"Alice"`, so `Object.is("Alice", "Alice")` is `true` and `ChildB` does not re-render. This demonstrates Zustand's precise subscription model vs Context.

</details>

---

### Q21

```jsx
// With Context
const Ctx = React.createContext();
function App() {
  const [val, setVal] = useState(0);
  return (
    <Ctx.Provider value={{ val, setVal }}>
      <DeepChild />
    </Ctx.Provider>
  );
}

// With Zustand
const useStore = create((set) => ({
  val: 0,
  setVal: (v) => set({ val: v }),
}));
function App() {
  return <DeepChild />;
}
```

What is the key structural difference in how `DeepChild` accesses `setVal`?

<details>
<summary>✅ Answer</summary>

```txt
Context: DeepChild must be a descendant of Ctx.Provider in the component tree.
         If App unmounts or the Provider is removed, DeepChild loses access.

Zustand: DeepChild imports useStore directly. No Provider ancestry required.
         The store is a global singleton and accessible anywhere.
```

**Explanation:** Context is tree-scoped. Zustand is module-scoped. This means Zustand stores can be accessed from completely separate React trees, React portals, outside React entirely (getState/subscribe), or in test utilities without wrapping with a Provider.

</details>

---

## 7. Edge Cases

### Q22

```js
const useStore = create((set) => ({
  count: 5,
}));

useStore.setState({ count: 5 }); // Same value
```

Does any subscribed component re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — components subscribed to `count` will re-render.
```

**Explanation:** This is a subtle difference from `useState`. In React's `useState`, setting the same value triggers a bail-out and prevents re-render. Zustand does NOT perform this optimization at the `set()` level — it always notifies subscribers when `set()` is called. The selector comparison then decides whether to re-render. Since the selector result is `5` both times, and `Object.is(5, 5)` is `true`, the component ultimately does **not** re-render. So the subscribers are notified, selectors run, but no actual DOM update occurs.

</details>

---

### Q23

```js
const useStore = create((set) => ({
  count: 0,
  reset: () => set({ count: 0 }, true), // Replace flag
}));

useStore.setState({ extraField: "hello" });
useStore.getState().reset();
console.log(useStore.getState());
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
{ count: 0 }
```

**Note:** Actions (like `reset`) are also lost because `true` replaces the entire state.

**Explanation:** `set({ count: 0 }, true)` replaces the **entire** store state — not just `count`. The `extraField`, `reset` action, and all other fields are wiped. Only `{ count: 0 }` remains. This is why using the replace flag carelessly is dangerous.

</details>

---

### Q24

```js
const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// Outside React component
const unsub = useStore.subscribe((state) => {
  console.log("subscriber:", state.count);
});

useStore.getState().increment();
useStore.getState().increment();
unsub(); // Unsubscribe
useStore.getState().increment();
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
subscriber: 1
subscriber: 2
```

**Explanation:** `subscribe` registers a callback that fires on every state change. The callback receives the full new state. After `unsub()`, the callback is deregistered and no longer fires. The third `increment` call does not log anything.

</details>

---

### Q25

```js
const useStore = create((set) => ({
  count: 0,
}));

const initialState = useStore.getState();
useStore.setState({ count: 99 });

// Store reset utility
useStore.setState(initialState, true);

console.log(useStore.getState().count);
```

What is logged?

<details>
<summary>✅ Answer</summary>

```txt
0
```

**Explanation:** `useStore.getState()` called before any `setState` captures the initial state `{ count: 0 }`. After setting `count` to `99`, `setState(initialState, true)` fully replaces the store state with the captured initial snapshot. `count` returns to `0`. This is the idiomatic pattern for store reset in tests and "logout" flows.

</details>

---

## ✅ Topics Covered

| Category | Questions | Concepts Tested |
|---|---|---|
| Basic Store | Q1–Q4 | Initial state, shallow merge, multiple component subscriptions |
| Selectors | Q5–Q8 | Selector precision, useShallow, no-selector pitfall |
| Actions | Q9–Q12 | Functional set, get() usage, async transitions, batching |
| Computed/Derived | Q13–Q15 | Selector computation, primitive comparison, length selector |
| Persist | Q16–Q18 | Rehydration, partialize, version migration |
| Zustand vs Context | Q19–Q21 | Context re-render behavior, Zustand precision, tree vs module scope |
| Edge Cases | Q22–Q25 | Same-value set, replace flag, external subscribe, store reset |
