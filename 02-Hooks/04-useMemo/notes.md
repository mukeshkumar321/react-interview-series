# React useMemo Hook

## Table of Contents

1. [What is useMemo](#1-what-is-usememo)
2. [Syntax and Basic Usage](#2-syntax-and-basic-usage)
3. [How useMemo Works Internally](#3-how-usememo-works-internally)
4. [When to Use useMemo](#4-when-to-use-usememo)
5. [When NOT to Use useMemo](#5-when-not-to-use-usememo)
6. [useMemo vs useCallback](#6-usememo-vs-usecallback)
7. [useMemo and Referential Equality](#7-usememo-and-referential-equality)
8. [useMemo for Expensive Derived Data](#8-usememo-for-expensive-derived-data)
9. [useMemo with React.memo](#9-usememo-with-reactmemo)
10. [Dependency Array Rules](#10-dependency-array-rules)
11. [Performance Measurement Before Memoizing](#11-performance-measurement-before-memoizing)
12. [Common Mistakes](#12-common-mistakes)
13. [Best Practices](#13-best-practices)

---

## 1. What is useMemo

`useMemo` is a React Hook that **memoizes the return value of a function** — it caches a computed result and only recomputes it when one of its listed dependencies changes.

### The Problem It Solves

Without `useMemo`, every render re-runs every computation inside the component body, even when the inputs to those computations haven't changed.

```jsx
function ProductList({ products, filterText }) {
  // This runs on EVERY render — even if products and filterText are unchanged
  const filtered = products.filter(p => p.name.includes(filterText));

  return filtered.map(p => <ProductCard key={p.id} product={p} />);
}
```

If `ProductList` re-renders because a parent's unrelated state changed, the filter runs unnecessarily over thousands of products. `useMemo` prevents that.

### The Mental Model

```text
Without useMemo:
  render → compute → result
  render → compute → result   (redundant if inputs unchanged)
  render → compute → result   (redundant if inputs unchanged)

With useMemo:
  render → compute → cached result
  render → deps same? → return cached result   (computation skipped ✅)
  render → deps changed? → recompute → new cached result
```

### What useMemo Is NOT

- Not a semantic guarantee — React may discard the cache in memory pressure scenarios
- Not a workaround for poor component design
- Not a substitute for `useCallback` (which memoizes functions, not values)
- Not required for all derived data — only for expensive computations or referential equality needs

---

## 2. Syntax and Basic Usage

### Syntax

```jsx
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

- The first argument is a "create" function — a function that computes and returns the value
- The second argument is a dependency array — React will recompute only when these change
- Returns the cached value on renders where deps are unchanged

### Basic Example

```jsx
import { useMemo, useState } from 'react';

function FilteredList({ items }) {
  const [filter, setFilter] = useState('');

  const filteredItems = useMemo(
    () => items.filter(item => item.name.toLowerCase().includes(filter.toLowerCase())),
    [items, filter]
  );

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <ul>
        {filteredItems.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
}
```

### When the Computation Runs

```text
Initial render:
  → useMemo calls the function and caches the result

Re-render with same deps:
  → useMemo returns the cached result (function NOT called)

Re-render with changed deps:
  → useMemo calls the function again and caches the new result
```

---

## 3. How useMemo Works Internally

### Dependency Comparison

React stores the previous dependency array alongside the memoized value in the component's fiber node. On each render:

```text
1. Read the new dependency array
2. Compare each element against the stored (previous) dependencies using Object.is
3. If ALL elements are equal → return cached value
4. If ANY element changed → call the create function → store new result and new deps → return new result
```

### Fiber Node Storage

```text
Component Fiber Node
  ↓
memoizedState (hook linked list)
  ↓
useMemo hook: {
  memoizedState: [cachedValue, previousDeps]
  next: ...
}
```

### Object.is Comparison

React uses `Object.is` for dependency comparison — the same algorithm used by `useEffect` and `useCallback`:

```jsx
// These deps are considered CHANGED (triggers recomputation):
const obj = { a: 1 };    // new object reference each render
const arr = [1, 2, 3];   // new array reference each render
const fn = () => {};     // new function reference each render

// These deps are considered SAME (cache preserved):
const num = 42;          // primitive, same value
const str = 'hello';     // primitive, same value
const bool = true;       // primitive, same value
```

### Not a Semantic Guarantee

```text
React may flush the cache in future versions for:
  - Memory pressure optimizations
  - React's own internal needs (like keeping offscreen trees lean)

This means useMemo is a performance hint, not a correctness guarantee.
Code must be correct without the cache — useMemo only adds performance.
```

---

## 4. When to Use useMemo

### Expensive Computations

Use `useMemo` when the computation takes noticeably long relative to the frequency of renders:

```jsx
function DataGrid({ data, sortConfig }) {
  // Sorting thousands of rows is expensive
  const sortedData = useMemo(() => {
    return [...data].sort((a, b) => {
      if (a[sortConfig.key] < b[sortConfig.key]) return sortConfig.dir === 'asc' ? -1 : 1;
      if (a[sortConfig.key] > b[sortConfig.key]) return sortConfig.dir === 'asc' ? 1 : -1;
      return 0;
    });
  }, [data, sortConfig]);

  return sortedData.map(row => <Row key={row.id} row={row} />);
}
```

### Referential Equality for Dependencies

When an object or array computed inside a component is used as a dependency in `useEffect`, `useCallback`, or another `useMemo`, memoizing it prevents unnecessary cascading re-runs:

```jsx
function SearchComponent({ query, userId }) {
  // Without useMemo: new object reference every render → useEffect runs every render
  const searchParams = useMemo(
    () => ({ query, userId, timestamp: Date.now() }),
    [query, userId]
  );

  useEffect(() => {
    fetchSearch(searchParams);
  }, [searchParams]); // stable reference ✅
}
```

### Referential Equality for React.memo Children

When a computed value is passed as a prop to a `React.memo`-wrapped child, memoizing it ensures the child doesn't re-render unnecessarily:

```jsx
const Chart = React.memo(({ data }) => {
  return <ExpensiveChart data={data} />;
});

function Dashboard({ rawData, theme }) {
  // Without useMemo: new array ref each render → Chart re-renders even if rawData unchanged
  const chartData = useMemo(
    () => rawData.map(point => ({ x: point.date, y: point.value })),
    [rawData]
  );

  return <Chart data={chartData} />;
}
```

### Decision Criteria

```text
Ask these questions before adding useMemo:
  1. Is the computation actually expensive? (> ~1ms)
  2. Does this value change frequently or rarely?
  3. Is this value passed to React.memo children or used in useEffect deps?
  4. Does profiling show this as a bottleneck?

If most answers are "yes" → use useMemo
If most answers are "no" → skip useMemo
```

---

## 5. When NOT to Use useMemo

### Premature Optimization

`useMemo` itself has a cost — it allocates memory for the cache, runs comparisons on every render, and adds cognitive overhead. For cheap computations, this cost outweighs the benefit.

❌ Wrong — memoizing trivial operations:

```jsx
// Don't do this:
const doubled = useMemo(() => count * 2, [count]);

// Just compute it:
const doubled = count * 2;
```

### Simple Array/Object Operations

```jsx
// Don't memoize simple transformations:
const title = useMemo(() => name.toUpperCase(), [name]);     // trivial
const styles = useMemo(() => ({ color: 'red' }), []);         // just define outside component
const config = useMemo(() => ({ timeout: 5000 }), []);        // constant — hoist it
```

### When the Value is Rarely Read

```jsx
// If this component rarely renders and the list is small:
const sortedItems = useMemo(() => items.sort(...), [items]);
// Not worth it — the sort is fast and renders are rare
```

### Dependency Instability

If the dependencies themselves change on every render, `useMemo` provides no benefit:

```jsx
function App({ data }) {
  const config = { threshold: 5 }; // new object every render

  // This recomputes every render because config is always a new reference
  const processed = useMemo(
    () => processData(data, config),
    [data, config]  // config is always "new" → always recomputes
  );
}
```

### Summary — When NOT to use

| Scenario | Reason to Skip |
|----------|----------------|
| Primitive arithmetic | Cheaper than memoization overhead |
| String operations | Virtually free |
| Small arrays (< 100 items) | Filter/sort is fast |
| Component renders infrequently | Rarely renders = rarely wastes time |
| Deps change every render | Memoization provides no benefit |
| Value not used in React.memo props or useEffect deps | No referential equality benefit |

---

## 6. useMemo vs useCallback

These two hooks are closely related. The key difference:

| Hook | Returns | Use For |
|------|---------|---------|
| `useMemo` | The **value** returned by the function | Memoizing computed values, objects, arrays |
| `useCallback` | The **function** itself | Memoizing event handlers, callbacks |

### They Are Equivalent

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`:

```jsx
// These are identical:
const memoizedCallback = useCallback(() => doSomething(x), [x]);
const memoizedCallback = useMemo(() => () => doSomething(x), [x]);
```

### When to Use Which

```jsx
// useMemo — returns a computed value
const sortedList = useMemo(() => [...items].sort(compareFn), [items]);
const filteredData = useMemo(() => data.filter(predicate), [data, predicate]);
const chartConfig = useMemo(() => buildChartConfig(theme, data), [theme, data]);

// useCallback — returns a function (stable reference)
const handleSubmit = useCallback((e) => {
  e.preventDefault();
  submitForm(formData);
}, [formData]);

const fetchData = useCallback(() => {
  fetch(`/api?id=${userId}`).then(setData);
}, [userId]);
```

### The Key Question

Ask: "Am I caching a **value** or a **function**?"

```text
Caching a value (number, string, object, array) → useMemo
Caching a function (event handler, fetch function) → useCallback
```

---

## 7. useMemo and Referential Equality

This is the most nuanced use case for `useMemo` — stabilizing object and array references.

### The Problem — Unstable References

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const [filter, setFilter] = useState('');

  // New object created on EVERY render
  const options = { threshold: 10, filter };

  return (
    <>
      <ExpensiveChild options={options} />
      <button onClick={() => setCount(c => c + 1)}>Increment (unrelated)</button>
    </>
  );
}
```

Every time `count` changes, `Parent` re-renders. Even if `filter` didn't change, `options` is a new object. If `ExpensiveChild` is wrapped in `React.memo`, it still re-renders because `options` has a new reference.

### The Fix — useMemo for Referential Stability

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const [filter, setFilter] = useState('');

  const options = useMemo(
    () => ({ threshold: 10, filter }),
    [filter]           // only recomputes when filter changes
  );

  return (
    <>
      <ExpensiveChild options={options} />
      <button onClick={() => setCount(c => c + 1)}>Increment (unrelated)</button>
    </>
  );
}
```

Now `options` is the same object reference when `filter` hasn't changed. `React.memo` on `ExpensiveChild` can successfully bail out.

### Cascading Effect — useMemo Stabilizing useEffect Deps

```jsx
function UserSearch({ userId }) {
  const [results, setResults] = useState([]);

  // Without useMemo: new object every render → useEffect runs every render
  const params = useMemo(
    () => ({ userId, page: 1, limit: 20 }),
    [userId]
  );

  useEffect(() => {
    fetchUsers(params).then(setResults);
  }, [params]); // stable reference — runs only when userId changes ✅

  return <ResultList results={results} />;
}
```

### Why Primitives Don't Need useMemo

```jsx
// Fine — primitives are compared by value, not reference
const doubled = count * 2;
useEffect(() => { ... }, [doubled]); // Object.is(4, 4) = true ✅

// Needs useMemo — objects compared by reference
const config = useMemo(() => ({ value: count }), [count]);
useEffect(() => { ... }, [config]); // stable reference ✅
```

---

## 8. useMemo for Expensive Derived Data

### Filtering Large Lists

```jsx
function ProductCatalog({ products, filters }) {
  const { category, minPrice, maxPrice, inStockOnly } = filters;

  const filteredProducts = useMemo(() => {
    console.time('filter');
    const result = products
      .filter(p => !category || p.category === category)
      .filter(p => p.price >= minPrice && p.price <= maxPrice)
      .filter(p => !inStockOnly || p.inStock);
    console.timeEnd('filter');
    return result;
  }, [products, category, minPrice, maxPrice, inStockOnly]);

  return (
    <div className="grid">
      {filteredProducts.map(p => <ProductCard key={p.id} product={p} />)}
    </div>
  );
}
```

### Complex Data Transformations

```jsx
function SalesReport({ rawData, groupBy, dateRange }) {
  const processedData = useMemo(() => {
    const filtered = rawData.filter(
      d => d.date >= dateRange.start && d.date <= dateRange.end
    );

    const grouped = filtered.reduce((acc, item) => {
      const key = item[groupBy];
      if (!acc[key]) acc[key] = { label: key, total: 0, count: 0 };
      acc[key].total += item.amount;
      acc[key].count += 1;
      return acc;
    }, {});

    return Object.values(grouped).sort((a, b) => b.total - a.total);
  }, [rawData, groupBy, dateRange]);

  return <BarChart data={processedData} />;
}
```

### Tree Transformation

```jsx
function TreeView({ flatItems }) {
  // Building a tree from flat data is O(n log n) — worth memoizing for large lists
  const treeData = useMemo(() => {
    const map = {};
    const roots = [];

    flatItems.forEach(item => {
      map[item.id] = { ...item, children: [] };
    });

    flatItems.forEach(item => {
      if (item.parentId) {
        map[item.parentId]?.children.push(map[item.id]);
      } else {
        roots.push(map[item.id]);
      }
    });

    return roots;
  }, [flatItems]);

  return <Tree nodes={treeData} />;
}
```

---

## 9. useMemo with React.memo

### How They Work Together

- `React.memo` wraps a component to skip re-rendering when props haven't changed (by shallow comparison)
- `useMemo` stabilizes the prop values passed to that component

Used together, they form a complete memoization strategy:

```text
React.memo → prevents component re-render when props are reference-equal
useMemo     → ensures complex prop values maintain reference equality when inputs haven't changed
```

### Example — Without Memoization (Bug)

```jsx
const UserList = React.memo(({ users }) => {
  console.log('UserList rendered');
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
});

function App() {
  const [count, setCount] = useState(0);
  const allUsers = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];

  // New array reference every render → React.memo fails to bail out
  const activeUsers = allUsers.filter(u => u.active !== false);

  return (
    <>
      <UserList users={activeUsers} />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </>
  );
}
```

`UserList rendered` prints on every button click — `React.memo` is ineffective because `activeUsers` is always a new array.

### Example — With Memoization (Correct)

```jsx
function App() {
  const [count, setCount] = useState(0);
  const allUsers = [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }];

  // Same reference when allUsers hasn't changed → React.memo can bail out
  const activeUsers = useMemo(
    () => allUsers.filter(u => u.active !== false),
    [allUsers]
  );

  return (
    <>
      <UserList users={activeUsers} />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </>
  );
}
```

Now clicking the count button does not re-render `UserList` because `activeUsers` retains its reference.

### Full Pattern — Multiple Layers

```jsx
// Layer 1: Memoized child component
const ExpensiveTable = React.memo(({ rows, onSelect }) => {
  return rows.map(row => (
    <TableRow key={row.id} row={row} onSelect={onSelect} />
  ));
});

function DataView({ rawData, userId }) {
  // Layer 2: Memoized value (stable array reference)
  const processedRows = useMemo(
    () => rawData.map(d => transformRow(d)),
    [rawData]
  );

  // Layer 3: Memoized callback (stable function reference)
  const handleSelect = useCallback((id) => {
    console.log('Selected:', id, 'by user', userId);
  }, [userId]);

  return <ExpensiveTable rows={processedRows} onSelect={handleSelect} />;
}
```

---

## 10. Dependency Array Rules

The dependency array in `useMemo` follows the same rules as `useEffect` and `useCallback`.

### Rules

1. Include every value from the component scope that the computation uses
2. Do not include values that are stable across renders (e.g., `setState` functions, `useRef` objects)
3. Do not include values computed from other deps (extract them or include the originals)

### Common Dependencies

```jsx
// Include: state values
const [count] = useState(0);
const result = useMemo(() => count * 2, [count]);

// Include: props
function Comp({ data }) {
  const processed = useMemo(() => process(data), [data]);
}

// Include: context values used in computation
const theme = useContext(ThemeContext);
const styles = useMemo(() => buildStyles(theme), [theme]);

// NOT required: setState (stable), refs (stable), module-level constants
const setCount = useState(0)[1]; // stable — omit from deps
const myRef = useRef(null);      // stable object — omit from deps
const CONSTANT = 42;             // module-level — omit from deps
```

### The Exhaustive Deps ESLint Rule

The `react-hooks/exhaustive-deps` ESLint rule warns when dependencies are missing. It should not be suppressed lightly.

```jsx
// ESLint warning: 'userId' is missing from deps
const params = useMemo(() => ({ query, userId }), [query]);
//                                          ^^^^^^ missing!

// Correct:
const params = useMemo(() => ({ query, userId }), [query, userId]);
```

### Object Dependencies and Instability

```jsx
function App({ config }) {
  // config is an object prop — new reference on every parent render
  const result = useMemo(() => compute(config.value), [config]);
  // This recomputes every time App re-renders if parent passes a new config object

  // Better: depend on the specific primitive value you actually use
  const result2 = useMemo(() => compute(config.value), [config.value]);
}
```

---

## 11. Performance Measurement Before Memoizing

### The Golden Rule

Measure before you optimize. Adding `useMemo` everywhere is not free — it adds memory and comparison overhead. Only memoize when you have evidence of a performance problem.

### Using console.time

```jsx
const processedData = useMemo(() => {
  console.time('processData');
  const result = expensiveProcess(rawData);
  console.timeEnd('processData');
  return result;
}, [rawData]);
```

If `processData: 0.1ms` — probably not worth memoizing.
If `processData: 50ms` — definitely memoize.

### Using React DevTools Profiler

```text
React DevTools Profiler steps:
1. Open DevTools → Profiler tab
2. Click "Record"
3. Interact with the component (click buttons, type, etc.)
4. Click "Stop"
5. Inspect the flame chart:
   - Look for components that render frequently
   - Look for long render times (yellow/red in flame chart)
   - Check "Why did this render?" for prop/state changes
6. If a component renders often with expensive computed props → add useMemo
```

### The Threshold Rule of Thumb

```text
Computation time < 1ms  → skip useMemo
Computation time 1-10ms → consider useMemo if component renders frequently
Computation time > 10ms → useMemo is likely beneficial
```

---

## 12. Common Mistakes

### Mistake 1 — Missing Dependencies

❌ Wrong — `userId` missing from deps, stale computation:

```jsx
const query = useMemo(
  () => buildQuery(searchText, userId),
  [searchText] // userId missing → stale userId in query
);
```

✅ Correct:

```jsx
const query = useMemo(
  () => buildQuery(searchText, userId),
  [searchText, userId]
);
```

### Mistake 2 — Memoizing Trivial Computations

❌ Wrong — unnecessary overhead:

```jsx
const fullName = useMemo(() => `${firstName} ${lastName}`, [firstName, lastName]);
const isVisible = useMemo(() => count > 0, [count]);
const doubled = useMemo(() => count * 2, [count]);
```

✅ Correct — just compute inline:

```jsx
const fullName = `${firstName} ${lastName}`;
const isVisible = count > 0;
const doubled = count * 2;
```

### Mistake 3 — Unstable Dependencies Defeating Memoization

❌ Wrong — object dep recreated every render:

```jsx
function App({ data }) {
  const options = { threshold: 5 }; // new object every render

  const result = useMemo(
    () => processData(data, options),
    [data, options] // options always new → always recomputes
  );
}
```

✅ Correct — hoist constant or memoize the dep:

```jsx
const OPTIONS = { threshold: 5 }; // module-level constant — stable

function App({ data }) {
  const result = useMemo(
    () => processData(data, OPTIONS),
    [data] // OPTIONS is stable — not needed in deps
  );
}
```

### Mistake 4 — Memoizing JSX Directly

❌ Wrong — memoizing JSX with useMemo is unusual and misleading:

```jsx
const listItems = useMemo(
  () => items.map(item => <Item key={item.id} item={item} />),
  [items]
);
```

✅ Correct — either use React.memo on the component, or just render inline:

```jsx
// If items doesn't change often, just render:
{items.map(item => <Item key={item.id} item={item} />)}

// If Item itself is expensive, memoize the component:
const Item = React.memo(({ item }) => { ... });
```

### Mistake 5 — Assuming useMemo Prevents All Re-renders

❌ Wrong mental model:

```jsx
// useMemo does NOT prevent re-renders of the component that uses it
// It only caches the computed value within that render
const heavy = useMemo(() => expensiveCalc(x), [x]);
// The component still re-renders — useMemo just skips the expensiveCalc call
```

To prevent a component from re-rendering, use `React.memo` on the component itself.

---

## 13. Best Practices

### When to Use useMemo

| Scenario | useMemo? |
|----------|----------|
| Filtering/sorting large lists (> 100 items) | Yes |
| Complex mathematical transformations | Yes |
| Building data structures (trees, maps) from flat data | Yes |
| Object/array passed to React.memo child | Yes |
| Object/array used in useEffect dependency | Yes |
| Object/array used in useCallback dependency | Yes |
| Simple arithmetic or string operations | No |
| Value used only in current component's JSX (no children) | Rarely |
| Constant value that never changes | No — hoist to module level |

### Core Principles

1. **Measure first, memoize second** — Profile before adding `useMemo`. Premature optimization adds complexity without benefit.

2. **Correctness before performance** — Code must work correctly without memoization. `useMemo` is purely a performance optimization that React may ignore.

3. **Stabilize references for hooks and memoized children** — The most reliable use case is preventing unstable object/array references from causing cascading re-runs in `useEffect` or busting `React.memo`.

4. **Keep deps accurate** — Missing dependencies lead to stale data bugs. Never suppress the ESLint exhaustive-deps warning without understanding why.

5. **Move expensive code out when possible** — If a computation doesn't depend on props or state, move it out of the component entirely (module scope). It runs once when the module loads, never again.

```jsx
// Instead of:
function App() {
  const PRIMES = useMemo(() => generatePrimesUpTo(1000), []); // empty deps
}

// Do this:
const PRIMES = generatePrimesUpTo(1000); // module level — runs once ever

function App() {
  // PRIMES is always available, never recomputed
}
```

6. **useMemo + React.memo + useCallback form a triad** — They work together. `React.memo` memoizes a component, `useMemo` memoizes values, `useCallback` memoizes functions. Use all three together when you need a truly optimized subtree.

### Quick Reference

```text
useMemo
  ├─ Syntax: useMemo(() => computation, [deps])
  ├─ Returns: cached result of computation
  ├─ Recomputes: when any dep changes (Object.is comparison)
  ├─ Use when: expensive computation OR referential stability needed
  ├─ Don't use when: trivial computation OR deps change every render
  └─ Related: useCallback (memoizes functions), React.memo (memoizes components)
```
