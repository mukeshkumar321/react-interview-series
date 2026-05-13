# React useCallback Hook

## Table of Contents

1. [What is useCallback](#1-what-is-usecallback)
2. [Why Functions Recreate on Every Render](#2-why-functions-recreate-on-every-render)
3. [Referential Equality Problem](#3-referential-equality-problem)
4. [useCallback + React.memo Pattern](#4-usecallback--reactmemo-pattern)
5. [Dependency Array](#5-dependency-array)
6. [useCallback vs useMemo](#6-usecallback-vs-usememo)
7. [useCallback for useEffect Dependencies](#7-usecallback-for-useeffect-dependencies)
8. [Performance Cost](#8-performance-cost)
9. [Common Anti-Patterns](#9-common-anti-patterns)
10. [Stale Closure in useCallback](#10-stale-closure-in-usecallback)
11. [useCallback with useReducer](#11-usecallback-with-usereducer)
12. [Common Mistakes](#12-common-mistakes)
13. [Best Practices](#13-best-practices)

---

## 1. What is useCallback

`useCallback` is a React Hook that returns a **memoized version of a callback function**. The memoized function is only recreated when one of the specified dependencies changes.

### Syntax

```jsx
const memoizedFn = useCallback(fn, [deps]);
```

- `fn` — the function you want to memoize
- `[deps]` — dependency array; the function is recreated only when any dep changes
- Returns the same function reference across renders if dependencies are unchanged

### Simple Example

```jsx
import { useState, useCallback } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  // Without useCallback — new function every render
  const handleClickRaw = () => {
    setCount(prev => prev + 1);
  };

  // With useCallback — same function reference across renders
  const handleClickMemo = useCallback(() => {
    setCount(prev => prev + 1);
  }, []); // empty deps — function never changes

  return <button onClick={handleClickMemo}>{count}</button>;
}
```

### What "Same Reference" Means

```jsx
function App() {
  const [x, setX] = useState(0);

  const fn1 = () => {};                   // new object every render
  const fn2 = useCallback(() => {}, []);  // same object every render

  // On re-render triggered by setX:
  // fn1 → new reference (0x002, 0x003, ...)
  // fn2 → same reference (0x001) as long as deps are unchanged
}
```

---

## 2. Why Functions Recreate on Every Render

### JavaScript Closures and Render Cycles

Every time a React component renders, the entire function body executes from top to bottom. Every variable and function defined inside the component is recreated from scratch on each call.

```jsx
function MyComponent({ userId }) {
  // This entire block runs on EVERY render

  const fetchUser = () => {              // ← new function object every render
    fetch(`/api/users/${userId}`);
  };

  const handleSubmit = (e) => {          // ← new function object every render
    e.preventDefault();
    saveData();
  };

  return <ChildComponent onFetch={fetchUser} />;
}
```

### Functions Are Objects in JavaScript

In JavaScript, functions are first-class objects. Two separately created functions are never `===` equal, even if they have identical source code:

```js
const a = () => {};
const b = () => {};

console.log(a === b); // false — different object references in memory
console.log(a === a); // true — same reference
```

This is the same behavior as plain objects:

```js
const obj1 = { x: 1 };
const obj2 = { x: 1 };
console.log(obj1 === obj2); // false — different references
```

### Why This Is a Problem for React

React uses referential equality (`===`) internally in three critical places:

| Place | Purpose |
|-------|---------|
| `React.memo` prop comparison | Decides if memoized child should re-render |
| `useEffect` dependency comparison | Decides if effect should re-run |
| `useMemo` dependency comparison | Decides if memoized value needs recomputing |

If a function is recreated every render and passed as a prop to a `React.memo` child, the child will always re-render because the function reference is always new — defeating the purpose of memoization.

---

## 3. Referential Equality Problem

### What is Referential Equality

Referential equality means two variables point to the **same object in memory**. JavaScript's `===` operator checks referential equality for objects, arrays, and functions.

```js
// Primitives — compared by value
1 === 1              // true
'hello' === 'hello'  // true

// Objects, arrays, functions — compared by reference (memory address)
{} === {}            // false — different addresses
[] === []            // false — different addresses
(() => {}) === (() => {}) // false — different addresses
```

### Why React.memo Fails Without useCallback

`React.memo` wraps a component and skips re-rendering if all props are shallowly equal. It uses `===` for the comparison. When a parent passes a callback as a prop and that callback is recreated every render, `React.memo` sees a new prop value on every render and re-renders the child unconditionally.

```jsx
const Child = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // Problem: new function reference every render
  const handleClick = () => console.log('clicked');

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Increment Parent</button>
      <Child onClick={handleClick} />
      {/* Child re-renders on every Parent render despite React.memo */}
    </div>
  );
}
```

### Trace Through the Equality Failure

```text
Render 1:
  handleClick_v1 = () => {} (address: 0x001)
  <Child onClick={handleClick_v1} />

User clicks "Increment Parent" → count = 1 → Parent re-renders

Render 2:
  handleClick_v2 = () => {} (address: 0x002) ← NEW object
  React.memo compares:
    prev.onClick (0x001) !== new.onClick (0x002)
    ↓
  Child re-renders ← React.memo bypassed
```

### The Fix with useCallback

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // useCallback: same function reference if deps are unchanged
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []); // empty array — never recreates

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Increment Parent</button>
      <Child onClick={handleClick} />
      {/* Child does NOT re-render when count changes */}
    </div>
  );
}
```

```text
Render 1:
  handleClick_v1 = useCallback(() => {}, []) (address: 0x001)
  <Child onClick={handleClick_v1} />

User clicks "Increment Parent" → count = 1 → Parent re-renders

Render 2:
  handleClick = still 0x001 (deps unchanged — useCallback returns same ref)
  React.memo compares:
    prev.onClick (0x001) === new.onClick (0x001)
    ↓
  Child SKIPS re-render ✅
```

---

## 4. useCallback + React.memo Pattern

This is the **primary use case** for `useCallback`. Neither hook alone fully solves unnecessary re-renders — you need both working together.

### The Three Steps

```text
Step 1: Wrap the child component with React.memo
Step 2: Wrap the callback passed to that child with useCallback
Step 3: Provide correct dependencies in the useCallback dep array
```

### Step 1 — Wrap Child with React.memo

```jsx
// ❌ Not memoized — always re-renders regardless of props
const Button = ({ onClick, label }) => {
  console.log(`Button "${label}" rendered`);
  return <button onClick={onClick}>{label}</button>;
};

// ✅ Memoized — skips re-render if props are shallowly equal
const Button = React.memo(({ onClick, label }) => {
  console.log(`Button "${label}" rendered`);
  return <button onClick={onClick}>{label}</button>;
});
```

### Step 2 — Wrap Handler with useCallback

```jsx
function TodoList() {
  const [todos, setTodos] = useState([]);
  const [text, setText] = useState('');

  // text is used inside — must be in deps
  const handleAddTodo = useCallback(() => {
    setTodos(prev => [...prev, { id: Date.now(), text }]);
    setText('');
  }, [text]);

  // no external deps — empty array
  const handleDeleteTodo = useCallback((id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  }, []);

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <Button onClick={handleAddTodo} label="Add" />
      {todos.map(todo => (
        <TodoItem key={todo.id} todo={todo} onDelete={handleDeleteTodo} />
      ))}
    </div>
  );
}
```

### Complete Working Example

```jsx
import { useState, useCallback, memo } from 'react';

// Step 1: memo wraps the child
const ProductCard = memo(({ product, onAddToCart }) => {
  console.log(`ProductCard "${product.name}" rendered`);
  return (
    <div>
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
});

function Shop() {
  const [cart, setCart] = useState([]);
  const [filter, setFilter] = useState('all');

  const products = [
    { id: 1, name: 'Keyboard', price: 99 },
    { id: 2, name: 'Mouse', price: 49 },
  ];

  // Step 2: useCallback stabilizes the handler
  const handleAddToCart = useCallback((productId) => {
    setCart(prev => [...prev, productId]);
  }, []); // setCart is stable, productId comes from argument — no deps needed

  return (
    <div>
      <select onChange={e => setFilter(e.target.value)}>
        <option value="all">All</option>
        <option value="sale">On Sale</option>
      </select>

      <p>Cart: {cart.length} items</p>

      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onAddToCart={handleAddToCart}
        />
        // ProductCard does NOT re-render when filter changes
        // because handleAddToCart is the same reference
      ))}
    </div>
  );
}
```

### Behavior Comparison

```text
WITHOUT useCallback + React.memo:
  Shop renders (filter changed)
    ↓
  handleAddToCart = new function object (0x003)
    ↓
  React.memo: prev (0x001) !== new (0x003)
    ↓
  ALL ProductCard components re-render unnecessarily

WITH useCallback + React.memo:
  Shop renders (filter changed)
    ↓
  handleAddToCart = same function (0x001) — deps unchanged
    ↓
  React.memo: prev (0x001) === new (0x001)
    ↓
  ProductCard components SKIP re-render ✅
```

---

## 5. Dependency Array

The dependency array in `useCallback` follows the same rules as `useEffect`. The ESLint rule `react-hooks/exhaustive-deps` enforces these rules.

### Rules

| Rule | Explanation |
|------|-------------|
| Include all values from the outer scope used inside the callback | State, props, variables, other functions |
| Exclude stable references | `setState` functions, `dispatch`, `ref.current` |
| Empty array `[]` — callback is created once and never recreates | Suitable for handlers with no external dependencies |
| No array — callback always recreates (rarely useful) | Equivalent to no memoization |

### Correct Dependency Examples

```jsx
function Component({ userId, onSuccess }) {
  const [data, setData] = useState(null);

  const fetchData = useCallback(async () => {
    const result = await fetch(`/api/${userId}`);  // uses userId
    const json = await result.json();
    setData(json);    // setData is stable — omit from deps
    onSuccess(json);  // onSuccess from props — must include
  }, [userId, onSuccess]); // ✅ all external values listed

  return <button onClick={fetchData}>Fetch</button>;
}
```

### Using Functional Updates to Reduce Dependencies

If a callback only reads state to compute the next state, replace it with a functional update. This removes the state variable from deps entirely.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  // ❌ count is in deps — new function reference every time count changes
  const increment = useCallback(() => {
    setCount(count + 1);
  }, [count]);

  // ✅ functional update — count is not captured, no need in deps
  const increment = useCallback(() => {
    setCount(prev => prev + 1);
  }, []); // stable — never changes regardless of count value

  return <button onClick={increment}>{count}</button>;
}
```

### Object and Array Dependencies

Objects and arrays created inline during render are new references every render. Adding them to deps defeats the purpose of `useCallback`.

```jsx
// ❌ Problem: options is recreated every render
function Bad({ userId }) {
  const options = { method: 'GET', cache: 'no-store' }; // new object every render

  const fetchUser = useCallback(() => {
    fetch(`/api/${userId}`, options); // options changes reference every render
  }, [userId, options]); // options changes → fetchUser changes every render
}

// ✅ Fix 1: move static object outside the component
const DEFAULT_OPTIONS = { method: 'GET', cache: 'no-store' }; // stable

function Good({ userId }) {
  const fetchUser = useCallback(() => {
    fetch(`/api/${userId}`, DEFAULT_OPTIONS);
  }, [userId]); // only userId can change
}

// ✅ Fix 2: memoize the object with useMemo
function Good2({ userId, cachePolicy }) {
  const options = useMemo(() => ({
    method: 'GET',
    cache: cachePolicy,
  }), [cachePolicy]);

  const fetchUser = useCallback(() => {
    fetch(`/api/${userId}`, options);
  }, [userId, options]); // options is now stable unless cachePolicy changes
}
```

---

## 6. useCallback vs useMemo

### Mathematical Equivalence

`useCallback` and `useMemo` are the same mechanism internally. The equivalence is:

```js
useCallback(fn, deps) === useMemo(() => fn, deps)
```

Both memoize a value between renders. The distinction is the type of value returned:

| Hook | What it returns | Primary use |
|------|----------------|-------------|
| `useCallback(fn, deps)` | The function `fn` itself | Stabilizing function references |
| `useMemo(() => value, deps)` | The result of calling `() => value` | Caching expensive computed values |

### Side-by-Side

```jsx
// useCallback — returns the function
const handleClick = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
// typeof handleClick === 'function'

// useMemo — equivalent way to memoize a function
const handleClickViaMemo = useMemo(() => {
  return () => { doSomething(a, b); };
}, [a, b]);
// typeof handleClickViaMemo === 'function' — functionally identical to above

// useMemo — its intended use: memoizing a computed value
const sortedList = useMemo(() => {
  return [...items].sort((a, b) => a.price - b.price);
}, [items]);
// typeof sortedList === 'object' (array)
```

### When to Use Which

```jsx
// ✅ useMemo — for expensive computations that return a value
const filteredAndSorted = useMemo(() => {
  return items
    .filter(item => item.active && item.category === category)
    .sort((a, b) => a.name.localeCompare(b.name));
}, [items, category]);

// ✅ useCallback — for functions passed as props or used as useEffect deps
const handleFilter = useCallback((newCategory) => {
  setCategory(newCategory);
}, []);

// ✅ useCallback — for functions returned from custom hooks
function useSorter(items) {
  const sort = useCallback((key) => {
    setSortKey(key);
  }, []);
  return { sort };
}
```

### Common Confusion to Avoid

```jsx
// ❌ Wrong tool: useMemo used to memoize a function
const fn = useMemo(() => {
  return (x) => x * multiplier;
}, [multiplier]);
// Works, but useCallback is more readable for this purpose

// ✅ Correct tool for memoizing a function
const fn = useCallback((x) => x * multiplier, [multiplier]);

// ❌ Wrong tool: useCallback used to memoize a computed value
const total = useCallback(() => items.reduce((s, i) => s + i.price, 0), [items]);
// total is a FUNCTION, not a number — caller must call total() to get value

// ✅ Correct tool for memoizing a value
const total = useMemo(() => items.reduce((s, i) => s + i.price, 0), [items]);
// total is a number
```

---

## 7. useCallback for useEffect Dependencies

### The Problem: Unstable Function as useEffect Dependency

When a function defined inside a component is listed in a `useEffect` dependency array, the effect will re-run on every render — because the function is a new reference every render.

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  // New function reference on every render
  const fetchUser = () => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  };

  useEffect(() => {
    fetchUser();
  }, [fetchUser]); // ❌ fetchUser changes every render → infinite loop
}
```

### The Infinite Loop Trace

```text
Render 1:
  fetchUser_v1 created (0x001)
  useEffect runs (fetchUser_v1 is new dep)
  setUser(data) → triggers re-render

Render 2:
  fetchUser_v2 created (0x002) ← new reference
  useEffect compares: 0x001 !== 0x002 → runs again
  setUser(data) → triggers re-render

Render 3: ... INFINITE LOOP
```

### Fix with useCallback

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  // Stable function — only recreates when userId changes
  const fetchUser = useCallback(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]); // ✅ fetchUser is stable unless userId changes

  useEffect(() => {
    fetchUser();
  }, [fetchUser]); // ✅ effect re-runs only when userId changes

  return user ? <div>{user.name}</div> : <p>Loading...</p>;
}
```

### Stable Flow After Fix

```text
Render 1 (userId = 'abc'):
  fetchUser_v1 created (0x001)
  useEffect runs (new dep on mount)
  setUser(data) → triggers re-render

Render 2 (userId = 'abc', user changed):
  fetchUser still 0x001 (userId unchanged → useCallback returns same ref)
  useEffect: 0x001 === 0x001 → SKIPS ✅

Render 3 (userId = 'xyz'):
  fetchUser_v2 created (0x002) ← userId changed → useCallback recreates
  useEffect: 0x001 !== 0x002 → runs ✅ (correct — new user to fetch)
```

### Custom Hook Pattern

This pattern is especially important when building custom hooks that expose functions, since callers frequently use them as `useEffect` dependencies.

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);

  const execute = useCallback(async () => {
    setLoading(true);
    const res = await fetch(url);
    const json = await res.json();
    setData(json);
    setLoading(false);
  }, [url]); // ✅ stable unless url changes

  useEffect(() => {
    execute();
  }, [execute]); // ✅ runs when url changes

  return { data, loading, refetch: execute }; // refetch is stable
}
```

---

## 8. Performance Cost

### useCallback Has Overhead

`useCallback` is not free. On every render, React must:

1. Allocate memory for the new function you pass in (the function is always created)
2. Compare the new dependency array with the previous one
3. Decide whether to return the old memoized function or the new one
4. Retain the old function in memory until it is replaced

```text
Every render with useCallback:
  new function created in memory (always happens)
    ↓
  dependency array compared element-by-element
    ↓
  if deps unchanged: return old function reference
  if deps changed:   return new function, discard old

Cost = memory + comparison work on every render
Benefit = skipping child re-render when deps unchanged
```

### When the Benefit Exceeds the Cost

```text
WORTH USING useCallback:
  ✅ Callback passed to a React.memo child that renders often
  ✅ Function used as a useEffect dependency
  ✅ Function passed into an expensive custom hook
  ✅ Function placed in a context value (prevents all consumers from re-rendering)
  ✅ Custom hook that returns functions for callers to use as deps

NOT WORTH USING useCallback:
  ❌ Inline handler on a native DOM element <button onClick={fn}>
  ❌ Function used only inside this component, never passed down
  ❌ Component that renders very rarely (setup screens, modals that open once)
  ❌ Function not involved in any memoization or effect dependency decision
```

### Measuring Before Optimizing

```jsx
// React DevTools Profiler workflow:
// 1. Open React DevTools → Profiler tab
// 2. Record user interaction
// 3. Inspect which components re-rendered and why
// 4. Only add useCallback if you see unnecessary child renders

// ❌ Over-optimized — adds overhead with zero benefit
function SimpleForm() {
  const [value, setValue] = useState('');

  const handleChange = useCallback((e) => {
    setValue(e.target.value);
  }, []); // not passed to memoized child — zero benefit

  return <input onChange={handleChange} />;
}

// ✅ Appropriately optimized — prevents measurable unnecessary re-renders
function DataGrid({ rows }) {
  const [sortConfig, setSortConfig] = useState({ key: 'id', dir: 'asc' });

  const handleSort = useCallback((columnKey) => {
    setSortConfig(prev =>
      prev.key === columnKey
        ? { key: columnKey, dir: prev.dir === 'asc' ? 'desc' : 'asc' }
        : { key: columnKey, dir: 'asc' }
    );
  }, []); // passed to memoized HeaderCell — prevents N column re-renders per sort click

  return <MemoizedHeader columns={columns} onSort={handleSort} />;
}
```

---

## 9. Common Anti-Patterns

### Anti-Pattern 1 — useCallback Without React.memo on Child

The most widespread mistake. `useCallback` stabilizes a function reference, but that stable reference only prevents re-renders if the receiving component uses `React.memo` to check prop equality.

```jsx
// ❌ Pointless: Child does not use React.memo
const Child = ({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
};

function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <Child onClick={handleClick} />;
  // Child still re-renders on every Parent render
  // React never checks props equality without React.memo
  // useCallback provided zero benefit
}

// ✅ Complete pattern — both hooks needed
const Child = React.memo(({ onClick }) => {
  console.log('Child rendered');
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <Child onClick={handleClick} />;
  // Child now skips re-render when Parent re-renders ✅
}
```

### Anti-Pattern 2 — Wrapping Inline Handlers on Native Elements

Native DOM elements (`<button>`, `<input>`, `<div>`) never use prop equality checks. They are not React components — there is no memoization to bypass.

```jsx
// ❌ Pointless — native <input> does not use React.memo
function Form() {
  const [value, setValue] = useState('');

  const handleChange = useCallback((e) => {
    setValue(e.target.value);
  }, []);

  return <input onChange={handleChange} />;
  // <input> re-renders unconditionally — useCallback added overhead, no benefit
}
```

### Anti-Pattern 3 — Wrapping Every Function Defensively

This is cargo-cult optimization. Adding `useCallback` everywhere adds memory pressure and cognitive overhead without benefit.

```jsx
// ❌ Unnecessary useCallback on every internal function
function Dashboard({ userId }) {
  const [data, setData] = useState(null);

  const loadData  = useCallback(() => fetchData(userId), [userId]);  // used only in useEffect
  const formatVal = useCallback((v) => v.toFixed(2), []);           // pure function, no re-render concern
  const isAdmin   = useCallback(() => role === 'admin', [role]);    // used only in this component
  const getLabel  = useCallback((key) => labels[key], [labels]);    // same

  useEffect(() => { loadData(); }, [loadData]); // the ONE case where useCallback helps

  // All others: no memoized children, no useEffect deps → pure overhead
}
```

### Anti-Pattern 4 — Missing Dependencies

Omitting a dependency creates a stale closure. The callback runs with outdated values.

```jsx
function Search({ query }) {
  const [results, setResults] = useState([]);

  // ❌ query is used inside but not in deps — stale closure
  const handleSearch = useCallback(async () => {
    const data = await searchAPI(query); // query is frozen at callback creation
    setResults(data);
  }, []); // query changes from 'react' to 'hooks' but callback still uses 'react'
}

// ✅ Include all external values
const handleSearch = useCallback(async () => {
  const data = await searchAPI(query);
  setResults(data);
}, [query]); // recreates when query changes — fresh closure
```

---

## 10. Stale Closure in useCallback

### What is a Stale Closure

A stale closure occurs when a `useCallback`-memoized function was created with certain values captured in its closure, and those values have since changed — but the function was not recreated because the dependency array did not include them.

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  // count = 0 is captured at creation time
  const logCount = useCallback(() => {
    console.log('Count is:', count); // always logs 0 — stale!
  }, []); // empty deps — never recreates, never updates its closure

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={logCount}>Log</button>
      {/* After incrementing to 5, clicking Log still prints "Count is: 0" */}
    </div>
  );
}
```

### Detailed Stale Closure Example

```jsx
// ❌ Classic stale closure in a delayed callback
function Counter() {
  const [count, setCount] = useState(0);

  const handleAlertCount = useCallback(() => {
    setTimeout(() => {
      alert(`Count: ${count}`); // count captured at callback creation
    }, 3000);
  }, []); // never recreates — count is permanently 0 in this closure

  return (
    <>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={handleAlertCount}>Alert in 3s</button>
    </>
  );
}
// Scenario:
// 1. Click "Alert in 3s"
// 2. Rapidly click "Increment" five times → count = 5
// 3. Alert fires: "Count: 0" ← stale value shown
```

### Fix 1 — Include count in Dependencies

```jsx
// ✅ count in deps — function recreates when count changes
const handleAlertCount = useCallback(() => {
  setTimeout(() => {
    alert(`Count: ${count}`); // fresh — count is current
  }, 3000);
}, [count]); // recreates every time count changes

// Trade-off: function is recreated on every count change
// If passed to a memoized child, child re-renders every count change
```

### Fix 2 — Functional Update (Best for State Reads)

When the only reason for reading state is to compute the next state, functional updates eliminate the need to capture state at all.

```jsx
// ✅ No state in deps — truly stable
const handleIncrement = useCallback(() => {
  setCount(prev => prev + 1); // prev is always the current value — no closure over count
}, []); // empty — never recreates

// This works because React calls (prev => prev + 1) with the latest state value
// at the time of the state update, not at the time the function was created
```

### Fix 3 — useRef for Always-Fresh Values (When You Can't Use Functional Update)

For cases where you need to read state inside a callback but cannot use a functional update (for example, reading state without modifying it), a ref can hold the latest value.

```jsx
function Timer() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  // Keep ref synchronized with state on every render
  useEffect(() => {
    countRef.current = count;
  });

  // Stable function that always reads the latest count via ref
  const logCount = useCallback(() => {
    console.log('Count is:', countRef.current); // always fresh
  }, []); // stable — no deps needed because ref is mutable

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={logCount}>Log Count</button>
    </div>
  );
}
```

### Fix Comparison

| Fix | Callback Stability | Complexity | When to Use |
|-----|--------------------|-----------|-------------|
| Add state to deps | Recreates on state change | Low | Simple cases, child doesn't need to be stable |
| Functional update | Stable (empty deps) | Low | When only modifying state based on previous value |
| useRef | Stable (empty deps) | Medium | When reading state without modifying it |

---

## 11. useCallback with useReducer

### dispatch Is Already Stable

React guarantees that the `dispatch` function returned by `useReducer` has a **stable identity** across renders — it never changes. You do not need to wrap it in `useCallback`.

```jsx
function TodoApp() {
  const [state, dispatch] = useReducer(reducer, { todos: [] });

  // ❌ Unnecessary — dispatch is already stable
  const handleAdd = useCallback((text) => {
    dispatch({ type: 'ADD_TODO', payload: text });
  }, [dispatch]); // dispatch never changes, so this is pointless overhead

  // ✅ Use dispatch directly or in a thin wrapper without useCallback
  const handleAdd = (text) => dispatch({ type: 'ADD_TODO', payload: text });

  return <AddForm onAdd={handleAdd} />;
}
```

### When to Use useCallback alongside useReducer

Use `useCallback` when the handler contains non-trivial logic beyond just dispatching:

```jsx
function TodoApp() {
  const [state, dispatch] = useReducer(reducer, { todos: [] });

  // ✅ useCallback justified here:
  // 1. Passed to a memoized child component
  // 2. Contains validation logic that reads from state
  const handleAddWithValidation = useCallback((text) => {
    const trimmed = text.trim();
    if (trimmed.length === 0) return;
    if (state.todos.some(t => t.text === trimmed)) return; // duplicate check
    dispatch({ type: 'ADD_TODO', payload: trimmed });
  }, [state.todos]); // state.todos is a real dependency

  return <MemoizedAddForm onAdd={handleAddWithValidation} />;
}
```

### Context + useReducer Pattern

The combination of Context and `useReducer` is a lightweight alternative to Redux. `dispatch` is stable, so it can be put in context without causing consumer re-renders.

```jsx
const TodoDispatchContext = createContext(null);
const TodoStateContext = createContext(null);

function TodoProvider({ children }) {
  const [state, dispatch] = useReducer(reducer, { todos: [] });

  // Separate contexts for state and dispatch
  // Components that only need dispatch won't re-render when state changes
  return (
    <TodoDispatchContext.Provider value={dispatch}>
      <TodoStateContext.Provider value={state}>
        {children}
      </TodoStateContext.Provider>
    </TodoDispatchContext.Provider>
  );
}

// Consumer that only uses dispatch — won't re-render on state changes
function AddButton() {
  const dispatch = useContext(TodoDispatchContext);
  return (
    <button onClick={() => dispatch({ type: 'ADD_TODO', payload: 'New Task' })}>
      Add
    </button>
  );
}
```

---

## 12. Common Mistakes

### Mistake 1 — Forgetting React.memo on the Child

```jsx
// ❌ useCallback has no effect without React.memo on the child
const Child = ({ onAction }) => <button onClick={onAction}>Act</button>;

function Parent() {
  const onAction = useCallback(() => doSomething(), []);
  return <Child onAction={onAction} />;
  // Child re-renders every time Parent does — React.memo never consulted
}
```

### Mistake 2 — Missing Dependencies Causing Stale Callback

```jsx
// ❌ formData is missing from deps — callback has stale form state
function EditForm({ onSave }) {
  const [formData, setFormData] = useState({ name: '', email: '' });

  const handleSubmit = useCallback(() => {
    onSave(formData); // formData captured at mount: { name: '', email: '' }
  }, [onSave]); // ❌ formData not listed — always submits empty object

  return (
    <form onSubmit={handleSubmit}>
      <input onChange={e => setFormData(p => ({ ...p, name: e.target.value }))} />
      <button type="submit">Save</button>
    </form>
  );
}

// ✅ Include all external values read inside the callback
const handleSubmit = useCallback(() => {
  onSave(formData);
}, [onSave, formData]);
```

### Mistake 3 — Using useCallback in React Server Components

Server Components in Next.js App Router execute on the server and do not re-render. `useCallback` (and all hooks) are meaningless there.

```jsx
// ❌ Invalid — hooks cannot be used in Server Components
// app/users/page.tsx (Server Component by default in App Router)
export default function UsersPage() {
  const handleSelect = useCallback((id) => {}, []); // runtime error
  return <UserList onSelect={handleSelect} />;
}

// ✅ Mark as Client Component to use hooks
'use client';
export default function UsersPage() {
  const handleSelect = useCallback((id) => {}, []);
  return <UserList onSelect={handleSelect} />;
}
```

### Mistake 4 — Object or Array Dependency Recreated on Every Render

```jsx
// ❌ config is a new object every render → handler changes every render
function Component({ theme }) {
  const config = { theme, debug: false }; // new reference every render

  const handler = useCallback(() => {
    applyConfig(config);
  }, [config]); // config changes → handler changes → child re-renders
}

// ✅ Memoize the object, or pass primitives directly
function Component({ theme }) {
  const config = useMemo(() => ({ theme, debug: false }), [theme]);

  const handler = useCallback(() => {
    applyConfig(config);
  }, [config]); // config now stable unless theme changes
}
```

---

## 13. Best Practices

### 1 — Profile Before Optimizing

Do not add `useCallback` speculatively. Use React DevTools Profiler to confirm unnecessary re-renders exist before adding memoization.

```text
Profiling workflow:
  Open DevTools → Profiler → Record
    ↓
  Perform the interaction you want to optimize
    ↓
  Stop recording → inspect render flamegraph
    ↓
  Identify components re-rendering without visible cause
    ↓
  Check if a function prop is the cause (different reference each render)
    ↓
  Add React.memo to child + useCallback to parent
    ↓
  Re-profile to verify improvement
```

### 2 — Always Pair with React.memo

`useCallback` without `React.memo` on the child is overhead with no benefit. Treat them as a unit.

```jsx
// ✅ Always the complete pair
const Child = React.memo(({ onAction }) => { ... });
const onAction = useCallback(() => { ... }, []);
```

### 3 — Prefer Functional Updates to Minimize Dependencies

Functional updates to setState read the current value at update time, not at callback creation time. This keeps the dependency array small and the callback stable.

```jsx
// ✅ Minimal deps — highly stable
const add = useCallback((n) => {
  setCount(prev => prev + n); // no external state captured
}, []);
```

### 4 — Always Memoize Functions Returned from Custom Hooks

Custom hook consumers may use your returned functions as `useEffect` dependencies. Wrapping them in `useCallback` inside the hook prevents unintentional infinite loops.

```jsx
// ✅ Custom hook best practice
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);

  const increment = useCallback(() => setCount(p => p + 1), []);
  const decrement = useCallback(() => setCount(p => p - 1), []);
  const reset      = useCallback(() => setCount(initial), [initial]);

  return { count, increment, decrement, reset };
}
```

### 5 — Enable the ESLint Rule

The `exhaustive-deps` rule catches missing and unnecessary dependencies automatically:

```bash
npm install eslint-plugin-react-hooks --save-dev
```

```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

### 6 — Summary Decision Table

| Scenario | Use useCallback? | Reason |
|----------|-----------------|--------|
| Callback prop to a `React.memo` child | ✅ Yes | Prevents child re-render |
| Function used as `useEffect` dependency | ✅ Yes | Prevents infinite loop |
| Function returned from a custom hook | ✅ Yes | Callers may use as dep |
| Function in a memoized context value | ✅ Yes | Prevents all consumers re-rendering |
| Inline handler on `<button onClick>` | ❌ No | Native elements ignore prop equality |
| Function used only inside this component | ❌ No | No memoization to benefit |
| `dispatch` from `useReducer` | ❌ No | Already stable |
| Any hook in a Server Component | ❌ No | Server Components do not re-render |

---
