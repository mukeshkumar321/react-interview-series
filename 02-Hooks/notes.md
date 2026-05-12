# React Hooks ⚛️

React Hooks are one of the most important concepts in modern React development.

Hooks completely changed how React applications are written by allowing functional components to manage:

- State
- Side Effects
- Refs
- Context
- Performance Optimizations
- Reusable Stateful Logic

Before Hooks, these features were mainly available in class components.

Hooks are not just APIs.

To truly understand Hooks, you must understand:

- Closures
- Render Cycles
- Fiber Architecture
- Reconciliation
- Referential Equality
- React Scheduling
- Hook Internals

---

## 📚 Table of Contents

- [1. Introduction to Hooks](#1-introduction-to-hooks)
- [2. Rules of Hooks](#2-rules-of-hooks)
- [3. Hook Internals](#3-hook-internals)
- [4. Performance Patterns](#4-performance-patterns)


## 1. Introduction to Hooks

### What are Hooks?

Hooks are special React functions that allow functional components to use React features like:

- State
- Lifecycle Logic
- Side Effects
- Refs
- Context
- Memoization

Example:

```js
const [count, setCount] = useState(0);
```

Hooks were introduced in React 16.8.

---

### Why Hooks?

Hooks were introduced to solve multiple problems in React applications.

---

### Problems with Class Components

### 1. Complex Lifecycle Methods

Class components often duplicated logic across lifecycle methods.

Example:

```js
class User extends React.Component {
  componentDidMount() {
    document.title = this.state.name;
  }

  componentDidUpdate() {
    document.title = this.state.name;
  }
}
```

Hooks solved this using `useEffect`.

```js
useEffect(() => {
  document.title = name;
});
```

---

### 2. Reusing Stateful Logic Was Difficult

Before Hooks, developers used:

- Higher Order Components (HOC)
- Render Props

These patterns often created deeply nested component trees.

Hooks introduced reusable stateful logic through Custom Hooks.

Example:

```js
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const resize = () => setWidth(window.innerWidth);

    window.addEventListener("resize", resize);

    return () => {
      window.removeEventListener("resize", resize);
    };
  }, []);

  return width;
}
```

---

### 3. `this` Binding Confusion

Class components required manual binding.

Example:

```js
this.handleClick = this.handleClick.bind(this);
```

Hooks completely removed the need for `this`.

---

### 4. Large Components Became Difficult to Maintain

Lifecycle logic was often split across multiple methods.

Hooks allow related logic to stay together.

---

## Hooks & Functional Components

Before Hooks, functional components were mostly used for UI rendering.

Example:

```js
function App() {
  return <h1>Hello</h1>;
}
```

After Hooks:

```js
function App() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

Functional components became fully capable React components.

---

## Hooks & Closures

Understanding closures is mandatory for understanding Hooks.

---

## What is a Closure?

A closure is a function that remembers variables from its lexical scope even after the outer function has finished execution.

Example:

```js
function outer() {
  let count = 0;

  return function inner() {
    console.log(count);
  };
}
```

The inner function remembers `count`.

---

## Hooks Depend Heavily on Closures

Example:

```js
function Counter() {
  const [count, setCount] = useState(0);

  function log() {
    console.log(count);
  }

  return <button onClick={log}>Log</button>;
}
```

The `log` function captures the `count` value from the render during which it was created.

---

## Stale Closures

One of the most important React interview topics.

Example:

```js
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setInterval(() => {
      console.log(count);
    }, 1000);
  }, []);

  return (
    <button onClick={() => setCount(count + 1)}>
      Increment
    </button>
  );
}
```

Output:

```txt
0
0
0
0
```

Why?

Because the effect captured the closure from the initial render.

---

## Render Cycles

Every React render creates:

- New Variables
- New Functions
- New Closures

Example:

```js
function App() {
  const [count, setCount] = useState(0);

  console.log("render");
}
```

Every state update triggers:

1. Re-render
2. New Execution
3. New Closure Snapshot

---

## Important Mental Model

Each render is isolated.

React does NOT mutate old variables.

Instead:

```txt
Render 1 → count = 0
Render 2 → count = 1
Render 3 → count = 2
```

Each render has its own snapshot.

---

## 2. Rules of Hooks

Hooks follow strict rules internally.

These rules exist because React tracks hooks using execution order.

---

## Rule 1 — Only Call Hooks at the Top Level

### ❌ Wrong

```js
if (show) {
  useEffect(() => {});
}
```

---

### ❌ Wrong

```js
for (let i = 0; i < 3; i++) {
  useState();
}
```

---

### ✅ Correct

```js
useEffect(() => {}, []);
```

---

## Why?

React relies on hook execution order.

---

## Rule 2 — Only Call Hooks Inside React Functions

Hooks can only be called inside:

- React Components
- Custom Hooks

---

### ✅ Correct

```js
function App() {
  useState();
}
```

---

### ✅ Correct

```js
function useCustom() {
  useEffect(() => {});
}
```

---

### ❌ Wrong

```js
function test() {
  useState();
}
```

---

## Why Hook Order Matters

React internally tracks hooks by position.

Example:

```js
useState();
useEffect();
useRef();
```

Internally:

```txt
hooks[0] → useState
hooks[1] → useEffect
hooks[2] → useRef
```

---

## Internal Indexing

Imagine:

```js
if (condition) {
  useEffect();
}

useRef();
```

First Render:

```txt
0 → useState
1 → useEffect
2 → useRef
```

Second Render:

```txt
0 → useState
1 → useRef ❌
```

Hook indexing breaks.

This is why hooks must always execute in the same order.

---

## 3. Hook Internals

Understanding internals separates beginner React developers from advanced frontend engineers.

---

## Fiber Architecture

React Fiber is React’s reconciliation engine.

It was introduced to support:

- Incremental Rendering
- Scheduling
- Prioritization
- Concurrent Rendering

---

## What is Fiber?

A Fiber is a JavaScript object representing a component.

Each component has its own Fiber node.

Example:

```txt
<App />
 ├── <Navbar />
 ├── <Sidebar />
 └── <Content />
```

Each component becomes a Fiber node.

---

## Why Fiber Exists

Old React reconciliation was synchronous.

Large updates blocked the main thread.

Fiber allows React to:

- Pause Work
- Resume Work
- Prioritize Updates

---

## Hook Linked List

Hooks are stored as linked lists on Fiber nodes.

Example:

```txt
Fiber Node
   ↓
Hook → Hook → Hook
```

Each hook stores:

- Memoized State
- Update Queue
- Dependencies

---

## Internal Hook Object

Simplified structure:

```js
const hook = {
  memoizedState: null,
  queue: null,
  next: null
};
```

---

## Current Hook Pointer

React tracks hooks during rendering using pointers.

Example pseudo-code:

```js
let currentHook = null;
```

As hooks execute:

```txt
useState → move pointer
useEffect → move pointer
useRef → move pointer
```

---

## Render Phase vs Commit Phase

React rendering has two major phases.

---

## Render Phase

React:

- Calls Components
- Calculates JSX
- Creates Fiber Tree
- Compares Changes

No DOM updates happen here.

This phase must stay pure.

---

## Commit Phase

React:

- Updates DOM
- Runs Effects
- Applies Refs

This phase mutates the UI.

---

## useState Internals

Example:

```js
const [count, setCount] = useState(0);
```

Internally React creates:

```js
hook = {
  memoizedState: 0,
  queue: []
}
```

---

## State Updates

Calling:

```js
setCount(1);
```

does NOT immediately update state.

React:

1. Pushes Update Into Queue
2. Schedules Re-render
3. Processes Queue During Next Render

---

## Functional Updates

Important interview topic.

```js
setCount(c => c + 1);
```

This avoids stale state problems because React passes the latest state.

---

## useEffect Internals

React stores:

- Dependencies
- Cleanup Function
- Effect Callback

Example:

```js
useEffect(() => {
  console.log(count);
}, [count]);
```

React performs shallow comparison:

```js
oldDeps === newDeps
```

If dependencies changed:

- Cleanup Runs
- Effect Re-runs

---

## Effect Timing

Effects run AFTER the commit phase.

Sequence:

```txt
Render Phase
↓
Commit Phase
↓
useEffect Runs
```

---

## Closure Snapshots

Every render creates new closures.

Example:

```js
function App() {
  const [count] = useState(0);

  function log() {
    console.log(count);
  }
}
```

Every render creates a NEW `log` function.

Each function remembers values from its render.

---

## Important Mental Model

Think of renders like snapshots.

```txt
Render 1 Snapshot
Render 2 Snapshot
Render 3 Snapshot
```

Closures belong to specific snapshots.

---

## 4. Performance Patterns

React performance optimization is largely about controlling re-renders and reference equality.

---

## React.memo

`React.memo` memoizes components.

Example:

```js
const Child = React.memo(function Child() {
  return <h1>Child</h1>;
});
```

React skips re-render if props are unchanged.

---

## Shallow Comparison

`React.memo` performs shallow comparison.

Example:

```js
{} !== {}
[] !== []
```

Even identical objects create new references.

---

## Problem Example

```js
<Child user={{ name: "John" }} />
```

This creates new object references every render.

Child re-renders.

---

## useMemo

`useMemo` memoizes values.

Example:

```js
const sorted = useMemo(() => {
  return items.sort();
}, [items]);
```

Useful for:

- Expensive Calculations
- Stable References

---

## Important Warning

Do NOT overuse `useMemo`.

Memoization also has cost.

---

## useCallback

`useCallback` memoizes functions.

Example:

```js
const handleClick = useCallback(() => {
  console.log("clicked");
}, []);
```

Useful when passing callbacks to memoized children.

---

## Referential Equality

One of the most important React concepts.

---

## Primitive Equality

```js
1 === 1
```

---

## Reference Equality

```js
{} === {} // false
[] === [] // false
```

Objects compare by reference, not value.

---

## Why This Matters

React dependency arrays use reference comparison.

Example:

```js
useEffect(() => {}, [{}]);
```

Runs every render because object reference changes.

---

## Context Optimization

Context updates can cause unnecessary re-renders.

Example:

```js
<ThemeContext.Provider value={{ theme }}>
```

This creates new object references.

All consumers re-render.

---

## Better Approach

```js
const value = useMemo(() => ({ theme }), [theme]);
```

---

## Lazy Loading

Lazy loading helps reduce initial bundle size.

Example:

```js
const Dashboard = React.lazy(() => import("./Dashboard"));
```

Used with:

```js
<Suspense fallback={<Loader />}>
  <Dashboard />
</Suspense>
```

Benefits:

- Faster Initial Load
- Smaller Bundle Size
- Better Performance

---

## Virtualization

Rendering thousands of DOM nodes is expensive.

Virtualization renders only visible items.

Libraries:

- react-window
- react-virtualized

Example:

```js
import { FixedSizeList } from "react-window";
```

Benefits:

- Reduced DOM Nodes
- Better Scrolling Performance
- Lower Memory Usage

---

## Summary

Hooks are deeply connected with:

- Closures
- Render Cycles
- Fiber Architecture
- Reconciliation
- Referential Equality
- Scheduling

Understanding Hooks internally is extremely important for:

- React Interviews
- Performance Optimization
- Debugging
- Advanced Frontend Engineering

React Hooks are not just APIs.

They are deeply tied to how React itself works internally.