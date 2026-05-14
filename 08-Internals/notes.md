# React Internals

## Table of Contents

- [1. React Architecture Overview](#1-react-architecture-overview)
- [2. React Element & JSX](#2-react-element--jsx)
- [3. Fiber Architecture](#3-fiber-architecture)
- [4. Rendering Phases](#4-rendering-phases)
- [5. Reconciliation](#5-reconciliation)
- [6. Diffing Algorithm](#6-diffing-algorithm)
- [7. Keys & Element Identity](#7-keys--element-identity)
- [8. Batching](#8-batching)
- [9. Scheduling & Priorities](#9-scheduling--priorities)
- [10. DevTools & Debugging](#10-devtools--debugging)

---

## 1. React Architecture Overview

React's modern architecture (v16+) uses an advanced system to manage rendering efficiently.

---

### Core Components

**React Core:** Maintains component tree, state, and props.

**ReactDOM:** Renders React components to the browser DOM.

**React Events:** Synthetic event system with event delegation.

---

### Three Main Phases

```
Render Phase (Reconciliation)
         ↓
Commit Phase (Update DOM)
         ↓
Layout & Passive Effects (Browser)
```

---

## 2. React Element & JSX

### React Element

A React Element is a plain JavaScript object describing what to render:

```js
{
  type: "h1",
  key: null,
  props: {
    className: "title",
    children: "Hello"
  }
}
```

---

### JSX Compilation

JSX is transpiled to function calls:

```jsx
// JSX
<h1 className="title">Hello</h1>

// Transpiled
React.createElement("h1", { className: "title" }, "Hello")
```

---

### Modern JSX Transform (React 17+)

React 17 introduced automatic JSX transformation:

```jsx
// No need to import React anymore
function App() {
  return <h1>Hello</h1>;
}
```

---

## 3. Fiber Architecture

Fiber is React's internal data structure representing each node in the component tree.

---

### What is a Fiber?

A Fiber object stores:

- Component type and instance
- Props, state, and hooks
- Parent, sibling, child references
- Work to be done
- Side effects to run

---

### Fiber Tree

React maintains a parallel Fiber tree alongside the component tree:

```
Component Tree    →    Fiber Tree
    App          →    App Fiber
   /   \              /      \
  Nav  Main    →   Nav Fiber Main Fiber
```

---

### Why Fibers?

**Problems solved:**

1. **Prioritization:** High-priority updates can interrupt low-priority work
2. **Pausable work:** Rendering can be paused and resumed
3. **Reusable work:** Work can be discarded if higher-priority update arrives
4. **Error boundaries:** Better error handling and recovery

---

## 4. Rendering Phases

React rendering has two phases: Render and Commit.

---

### Render Phase (Reconciliation)

The Render Phase is where React:

1. Traverses the Fiber tree
2. Calls component functions
3. Creates/updates Fibers
4. Calculates what changed

**Important:** This phase doesn't mutate the DOM and can be paused or restarted.

---

### Commit Phase

The Commit Phase applies changes:

1. Updates the Real DOM
2. Runs lifecycle methods
3. Updates refs
4. Runs effects

**Important:** This phase is synchronous and cannot be interrupted.

---

### Render Phase Steps

```
Start Render
   ↓
Process Fiber
   ↓
Call Component Function
   ↓
Reconcile with Previous Fiber
   ↓
Create Effects List
   ↓
Move to Next Fiber
   ↓
Complete Work
```

---

### Commit Phase Steps

```
Before Mutations
   ↓
Mutations (Update DOM)
   ↓
Layout Phase
   ↓
Passive Effects (useEffect)
```

---

## 5. Reconciliation

Reconciliation is the process of determining what changed between renders.

---

### Reconciliation Rules

**Rule 1: Different Element Types**

If the element type changes, React destroys the old tree and creates a new one:

```jsx
// Before
<div>Content</div>

// After
<span>Content</span>
```

React destroys the entire div subtree.

---

**Rule 2: Same Element Type with Key**

If elements are the same type and have keys, React matches by key:

```jsx
// Before
<li key="a">A</li>
<li key="b">B</li>

// After
<li key="b">B</li>
<li key="a">A</li>
```

React reorders them without destroying.

---

## 6. Diffing Algorithm

React uses a heuristic-based algorithm to efficiently compare trees.

---

### Algorithm Heuristics

**1. Compare by type:** Different types = different trees.

```js
if (oldElement.type !== newElement.type) {
  // Destroy old tree, create new
}
```

**2. Use keys for lists:** Keys help match elements.

```js
for (let newChild of newChildren) {
  const oldChild = childrenByKey[newChild.key];
  // Reconcile newChild with oldChild
}
```

**3. Compare depth-first:** Don't recursively compare deep trees.

---

### Time Complexity

Traditional tree diffing: **O(n³)**

React's optimized diffing: **O(n)**

React achieves O(n) through:

- One-level-at-a-time comparison
- Key-based matching in lists
- Assumption that siblings don't move across levels

---

## 7. Keys & Element Identity

### Purpose of Keys

Keys tell React which items changed, were added, or removed:

```jsx
{items.map(item => (
  <li key={item.id}>{item.text}</li>
))}
```

---

### How Keys Work

React uses keys to match old and new elements:

```
Old: key="a" key="b" key="c"
     Alice    Bob     Charlie

New: key="c" key="a" key="b"
     Charlie Alice   Bob

React reorders without recreating DOM nodes.
```

---

### Key Matching Process

```jsx
// Before
<li key="1">A</li>
<li key="2">B</li>

// After
<li key="2">B</li>
<li key="1">A</li>

// React matches by key:
// - key="1" still exists, reuse its DOM node
// - key="2" still exists, reuse its DOM node
// - Just reorder them in the DOM
```

---

### Why Index Keys Are Problematic

```jsx
{items.map((item, index) => (
  <li key={index}>{item}</li>
))}
```

Problems:

- If list is filtered, index doesn't match the item
- If items are reordered, component state gets mixed up
- Performance degrades

---

### Best Practices for Keys

✅ Use stable unique IDs:

```jsx
key={item.id}
```

❌ Don't use index in dynamic lists:

```jsx
key={index}  // Problems!
```

❌ Don't generate random keys:

```jsx
key={Math.random()}  // Wrong!
```

---

## 8. Batching

Batching groups multiple state updates into a single re-render.

---

### Automatic Batching in Event Handlers

```jsx
const handleClick = () => {
  setCount(c => c + 1);
  setName("React");
  // React re-renders once with both updates
};
```

---

### Batching in React 18

React 18 batches updates in more scenarios:

```jsx
// Event handler (batched)
<button onClick={() => {
  setCount(c => c + 1);
  setName("React");
}} />

// Promise (batched in v18, not v17)
fetch('/api').then(() => {
  setCount(c => c + 1);
});

// setTimeout (batched in v18, not v17)
setTimeout(() => {
  setCount(c => c + 1);
}, 0);
```

---

### Opting Out of Batching

Use `flushSync` for immediate updates:

```jsx
import { flushSync } from 'react-dom';

const handleClick = () => {
  flushSync(() => {
    setCount(c => c + 1);
  });
  // count is updated synchronously here
  console.log(count);
};
```

---

## 9. Scheduling & Priorities

React uses a scheduler to prioritize updates based on urgency.

---

### Update Priorities

**Immediate:** User interactions (clicks), errors

**High:** Component updates from effects

**Normal:** Data fetching, setTimeout

**Low:** Non-urgent updates

---

### How Scheduling Works

```
Update Arrives
   ↓
Assign Priority
   ↓
Scheduler Queues Work
   ↓
High Priority? → Interrupt current work
   ↓
Process Update
```

---

### Example: Interrupting Work

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [data, setData] = useState(null);

  const handleClick = () => {
    // High priority - user input
    setCount(count + 1);
  };

  useEffect(() => {
    // Low priority - background update
    fetchData().then(setData);
  }, []);

  return (
    <button onClick={handleClick}>
      Count: {count}
    </button>
  );
}
```

When clicking, React prioritizes count update even if data is still loading.

---

## 10. DevTools & Debugging

### React DevTools Browser Extension

Provides:

- Component tree inspection
- Props and state inspection
- Performance profiling
- Re-render tracking
- Hook inspection

---

### Profiler API

Measure component rendering performance:

```jsx
import { Profiler } from 'react';

function onRender(id, phase, actualDuration, baseDuration) {
  console.log(`${id} (${phase}) took ${actualDuration}ms`);
}

<Profiler id="App" onRender={onRender}>
  <App />
</Profiler>
```

---

### React.StrictMode

Development-only mode that highlights issues:

```jsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

Detects:

- Unsafe lifecycle methods
- Unexpected side effects
- Missing dependencies
- Deprecated patterns

---

### Performance Issues

**Unnecessary re-renders:**

- Parent re-renders trigger child re-renders
- Use `React.memo` to prevent

**Large component trees:**

- Split into smaller components
- Use code splitting

**Expensive computations:**

- Use `useMemo` to memoize
- Move to worker threads

**Memory leaks:**

- Clean up effects
- Remove event listeners
- Cancel async operations

---

### Debugging Techniques

**Check render count:**

```jsx
useEffect(() => {
  console.count('render');
});
```

**Profile code:**

```js
console.time('operation');
// code...
console.timeEnd('operation');
```

**Use React DevTools Profiler:**

- Identify slow components
- Find unnecessary re-renders
- Measure rendering time

---

## Performance Best Practices

1. **Profile before optimizing** - Use DevTools Profiler
2. **Avoid premature optimization** - Only optimize when needed
3. **Use proper keys** - Helps reconciliation efficiency
4. **Memoize strategically** - Don't memoize everything
5. **Code split** - Load code on demand
6. **Monitor bundle size** - Use webpack-bundle-analyzer
7. **Lazy load routes** - Use React.lazy
