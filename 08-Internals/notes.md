#  React Internals

##  Table of Contents

1. [How React Works](#1-how-react-works)
2. [JSX Internals](#2-jsx-internals)
3. [Virtual DOM](#3-virtual-dom)
4. [Reconciliation](#4-reconciliation)
5. [Fiber Architecture](#5-fiber-architecture)
6. [Render vs Commit Phase](#6-render-vs-commit-phase)
7. [Scheduler](#7-scheduler)
8. [Batching](#8-batching)
9. [Hook Internals](#9-hook-internals)
10. [Concurrent Rendering](#10-concurrent-rendering)
11. [Suspense & Hydration](#11-suspense--hydration)

---

##  1. How React Works

React is a JavaScript UI library that converts components into efficient DOM updates.

Most developers think React directly updates the DOM whenever state changes.

That is NOT what actually happens internally.

React works in multiple stages.

Complete React internal pipeline:

```txt
JSX
 ↓
React.createElement()
 ↓
React Elements
 ↓
Virtual DOM
 ↓
Fiber Tree
 ↓
Render Phase
 ↓
Diffing
 ↓
Reconciliation
 ↓
Commit Phase
 ↓
Real DOM Update
```

---

###  Step 1 — JSX is Written

Example:

```jsx
function App() {
  return <h1>Hello React</h1>;
}
```

Browser cannot understand JSX directly.

JSX is transformed into JavaScript.

---

###  Step 2 — JSX Becomes React Elements

Babel converts JSX into:

```js
React.createElement(
  "h1",
  null,
  "Hello React"
);
```

This creates a React Element object.

Example:

```js
{
  type: "h1",
  props: {
    children: "Hello React"
  }
}
```

React Elements are:
- plain JavaScript objects
- immutable
- lightweight UI descriptions

They are NOT DOM nodes.

---

###  Step 3 — Virtual DOM Tree Creation

React converts React Elements into a Virtual DOM tree.

Example:

```txt
App
 └── h1
      └── "Hello React"
```

This tree exists entirely in memory.

---

###  Step 4 — Fiber Tree Creation

React creates Fiber Nodes from the Virtual DOM.

Fiber is React's internal data structure.

Each component becomes a Fiber Node.

Example:

```txt
App Fiber
 └── h1 Fiber
```

---

###  Step 5 — Render Phase Starts

React starts calculating:
- what changed
- what needs updating
- which components re-render

This phase:
- can pause
- can restart
- can abort

NO DOM updates happen here.

---

###  Step 6 — Reconciliation & Diffing

React compares:
- old Virtual DOM
- new Virtual DOM

Then determines:
- additions
- deletions
- updates

---

###  Step 7 — Commit Phase

After calculations complete:

React updates the Real DOM.

This phase:
- cannot pause
- is synchronous

---

###  Step 8 — Browser Paint

Finally browser:
- recalculates layout
- repaints screen

User sees updated UI.

---

##  React Mental Model

Think of React as:

```txt
React =
Rendering Engine
+ Scheduler
+ Reconciliation Engine
+ State Manager
```

---

##  2. JSX Internals

JSX is NOT HTML.

JSX is syntax sugar over JavaScript.

---

###  JSX Example

```jsx
const element = <h1>Hello</h1>;
```

Babel transforms it into:

```js
const element = React.createElement(
  "h1",
  null,
  "Hello"
);
```

---

###  React.createElement Structure

```js
React.createElement(
  type,
  props,
  children
);
```

Example:

```js
React.createElement(
  "button",
  {
    className: "btn"
  },
  "Click"
);
```

---

###  JSX Compilation

Compilation happens using:
- Babel
- TypeScript compiler
- SWC

Browser never sees JSX directly.

---

###  JSX Returns Objects

JSX becomes:

```js
{
  type: "button",
  props: {
    className: "btn",
    children: "Click"
  }
}
```

---

###  Why JSX Exists

Without JSX:

```js
React.createElement(
  "div",
  null,
  React.createElement(
    "h1",
    null,
    "Hello"
  )
);
```

Very hard to read.

JSX improves readability.

---

##  3. Virtual DOM

Virtual DOM is a lightweight JavaScript representation of the UI.

It is NOT the actual browser DOM.

---

##  Real DOM Problem

Direct DOM updates are expensive.

Browser operations include:
- layout calculation
- repaint
- reflow

Frequent updates hurt performance.

---

##  Virtual DOM Solution

React creates a virtual copy in memory.

Example:

```txt
Virtual DOM

App
 ├── Header
 ├── Sidebar
 └── Content
```

React updates memory first instead of browser DOM.

---

##  How Virtual DOM Works

Example:

```jsx
<h1>Hello</h1>
```

becomes:

```js
{
  type: "h1",
  props: {
    children: "Hello"
  }
}
```

React compares:
- old tree
- new tree

Then updates only changed parts.

---

##  Benefits of Virtual DOM

###  Efficient Updates

Only changed nodes updated.

---

###  Faster Comparisons

JavaScript object comparison is faster than DOM manipulation.

---

###  Declarative UI

Developer describes:
- WHAT UI should look like

React handles:
- HOW updates happen

---

##  Important Misconception

Virtual DOM itself is NOT the optimization.

The real optimization is:
- diffing
- reconciliation
- batching
- efficient scheduling

---

##  4. Reconciliation

Reconciliation is React’s process of:
- comparing trees
- finding differences
- determining minimal updates

---

##  Why Reconciliation Exists

Re-rendering entire DOM is expensive.

React instead:
- compares trees
- updates only changed nodes

---

##  Diffing Algorithm

React uses heuristics.

Full tree comparison complexity:

```txt
O(n³)
```

React optimized it to approximately:

```txt
O(n)
```

using assumptions.

---

##  Rule 1 — Different Element Types

```jsx
<div />
<span />
```

React destroys old subtree completely.

---

##  Rule 2 — Same Element Types

```jsx
<div className="a" />
<div className="b" />
```

React updates only changed attributes.

---

##  Rule 3 — Component Types

```jsx
<App />
```

Same component:
- preserve state

Different component:
- remount component

---

##  Rule 4 — Keys

Keys help React identify list items.

Good:

```jsx
items.map(item => (
  <Item key={item.id} />
))
```

Bad:

```jsx
items.map((item, index) => (
  <Item key={index} />
))
```

---

##  Why Index Keys Are Problematic

When list order changes:
- React mismatches items
- state bugs occur
- unnecessary re-renders happen

---

##  5. Fiber Architecture

Fiber is React’s internal reconciliation engine.

Introduced in:
- React 16

---

##  Why Fiber Was Introduced

Old React Stack Reconciler was:
- synchronous
- blocking
- non-interruptible

Large rendering operations could freeze UI.

---

##  Fiber Solves This

Fiber enables:
- interruptible rendering
- prioritization
- scheduling
- concurrent rendering

---

##  Fiber Node Structure

Simplified Fiber:

```js
const fiber = {
  type,
  stateNode,
  child,
  sibling,
  return,
  alternate,
  pendingProps,
  memoizedProps,
  memoizedState,
}
```

---

##  Important Fiber Links

###  child

First child.

---

###  sibling

Next sibling.

---

###  return

Parent node.

---

##  Fiber Tree Example

```txt
App
 ├── Header
 ├── Sidebar
 └── Content
```

Internally:

```txt
Fiber(App)
  ↓ child
Fiber(Header)
  ↓ sibling
Fiber(Sidebar)
  ↓ sibling
Fiber(Content)
```

---

##  Work In Progress Tree

React maintains:

```txt
1. Current Tree
2. Work In Progress Tree
```

Updates happen on:
- Work In Progress Tree

After completion:
- trees swap

This technique is called:

```txt
Double Buffering
```

---

##  Fiber Enables Interruption

React can:
- pause rendering
- resume later
- prioritize urgent updates

---

##  6. Render vs Commit Phase

React rendering has 2 phases.

---

##  Render Phase

Purpose:
- calculate updates
- create Work In Progress tree
- diff Virtual DOM

NO DOM updates happen here.

---

##  Render Phase Characteristics

Render phase:
- asynchronous
- interruptible
- restartable
- pure

---

##  Commit Phase

Purpose:
- update DOM
- execute effects
- attach refs

---

##  Commit Phase Characteristics

Commit phase:
- synchronous
- non-interruptible

---

##  Internal Flow

```txt
State Update
 ↓
Render Phase
 ↓
Diffing
 ↓
Reconciliation
 ↓
Commit Phase
 ↓
DOM Updates
```

---

##  Important Rule

Never cause side effects during render.

Bad:

```jsx
fetch("/api");
```

inside component body.

---

##  7. Scheduler

Scheduler determines:
- when work runs
- which work is important

---

##  Why Scheduler Exists

Not all updates are equally important.

Example:

```txt
Typing Input → High Priority
Analytics Logging → Low Priority
```

---

##  Scheduler Responsibilities

Scheduler:
- prioritizes work
- pauses work
- resumes work
- avoids blocking UI

---

##  Priority Levels

Examples:

```txt
Immediate Priority
User Blocking Priority
Normal Priority
Low Priority
Idle Priority
```

---

##  React Lanes

React internally uses:
- lanes

Lanes represent update priorities.

---

##  Example

```txt
Typing → Sync Lane
Transition → Transition Lane
```

---

##  Scheduler Goal

Keep UI responsive.

---

##  8. Batching

React batches multiple state updates together.

---

##  Example

```jsx
setCount(c => c + 1);
setCount(c => c + 1);
```

Result:

```txt
+2
```

---

##  Stale Closure Example

```jsx
setCount(count + 1);
setCount(count + 1);
```

Result:

```txt
+1
```

because both updates capture same value.

---

##  Why Batching Exists

Without batching:

```txt
Update
Render
Update
Render
Update
Render
```

Too expensive.

---

##  With Batching

```txt
Multiple Updates
 ↓
Single Render
```

Efficient.

---

##  Automatic Batching

React 18 introduced broader automatic batching.

Works across:
- promises
- timeouts
- async operations

---

##  9. Hook Internals

Hooks are internally stored on Fiber nodes.

---

##  Hook Order Matters

React identifies hooks by:
- execution order

Bad:

```jsx
if (condition) {
  useState();
}
```

This breaks hook order.

---

##  Internal Hook Storage

Simplified:

```txt
Fiber
 ├── Hook1
 ├── Hook2
 └── Hook3
```

---

##  useState Internals

Simplified idea:

```js
function useState(initialValue) {
  const hook = getCurrentHook();

  return [
    hook.state,
    hook.dispatch
  ];
}
```

---

##  State Update Queue

Each hook has:
- update queue

Example:

```txt
Update1 → Update2 → Update3
```

React processes queue during render.

---

##  useEffect Internals

Effects run AFTER commit phase.

Flow:

```txt
Render
 ↓
Commit
 ↓
Effect Runs
```

Cleanup flow:

```txt
Old Cleanup
 ↓
New Effect
```

---

##  useMemo Internals

React stores:
- cached value
- dependency array

If dependencies unchanged:
- return cached value

---

##  useCallback Internals

`useCallback` memoizes:
- function reference

Prevents unnecessary child renders.

---

##  10. Concurrent Rendering

Concurrent Rendering allows React to:
- pause work
- interrupt work
- prioritize updates

---

##  Old React Problem

Large rendering tasks blocked UI.

Example:
- typing lag
- frozen animations

---

##  Concurrent Rendering Solution

React can:
- stop rendering
- process urgent update
- continue later

---

##  Example

```txt
Large List Rendering
        ↓ interrupted
User Typing
        ↓ processed immediately
Resume List Rendering
```

---

##  startTransition

React provides:

```jsx
startTransition(() => {
  setSearchResults(data);
});
```

Marks update as:
- non-urgent

---

##  Benefits

Concurrent rendering improves:
- responsiveness
- smooth UI
- user experience

---

##  Important Note

Concurrent Rendering is NOT multi-threading.

JavaScript still runs on:
- single thread

React simply schedules work smarter.

---

##  11. Suspense & Hydration

##  Suspense

Suspense allows React to:
- pause rendering
- show fallback UI
- continue later

---

##  Example

```jsx
<Suspense fallback={<Loader />}>
  <Dashboard />
</Suspense>
```

---

##  Suspense Flow

```txt
Component Suspends
 ↓
Fallback UI Appears
 ↓
Data Resolves
 ↓
Component Continues Rendering
```

---

##  Hydration

Hydration attaches React logic to server-rendered HTML.

---

##  Why Hydration Exists

Server-side rendering produces:
- static HTML

Hydration makes it interactive.

---

##  Hydration Flow

```txt
Server Rendered HTML
 ↓
Browser Loads JS
 ↓
Hydration
 ↓
Interactive React App
```

---

##  Hydration Mismatch

Mismatch occurs when:
- server output differs from client output

Example:

```jsx
Math.random()
Date.now()
```

during render.

---

##  Streaming Hydration

Modern React supports:
- progressive hydration
- selective hydration
- streaming HTML

Improves performance.

---

##  Final React Internal Flow

```txt
JSX
 ↓
createElement
 ↓
React Elements
 ↓
Virtual DOM
 ↓
Fiber Tree
 ↓
Scheduler
 ↓
Render Phase
 ↓
Diffing
 ↓
Reconciliation
 ↓
Commit Phase
 ↓
DOM Updates
 ↓
Effects
```

---

##  Final Mental Model

Think of React as:

```txt
React =
UI Engine
+ Scheduler
+ Reconciliation Engine
+ State Manager
+ Concurrent Rendering System
```

React continuously:
- builds trees
- compares trees
- schedules updates
- prioritizes rendering
- synchronizes UI efficiently