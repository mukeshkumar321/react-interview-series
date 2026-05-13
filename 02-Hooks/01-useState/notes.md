# React `useState` Hook

> The `useState` hook is the most commonly asked React Hook in interviews.  
> It allows functional components to store and update state.

---

## 📚 Table of Contents

1. Introduction to `useState`
2. Why `useState`?
3. Syntax
4. How `useState` Works Internally
5. State Update Process
6. Functional Updates
7. Batching in React
8. Lazy Initialization
9. Object & Array State Updates
10. Common Mistakes
11. Important Interview Concepts
12. Real-World Examples
13. Performance Considerations
14. Interview Questions
15. Summary

---

## 1 Introduction to `useState`

Before Hooks, state was only available in **Class Components**.

React introduced Hooks in React 16.8 so that Functional Components could also:
- manage state
- use lifecycle logic
- reuse logic

`useState` is the simplest Hook.

---

## 2 Why `useState`?

Without `useState`, functional components were just UI functions.

```jsx
function App() {
  return <h1>Hello</h1>;
}
```

They could not:
- store data
- track user input
- re-render dynamically

`useState` solved this problem.

---

## 3️ Syntax

```jsx
const [state, setState] = useState(initialValue);
```

---

### Example

```jsx
import React, { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

---

### Breakdown

| Part | Meaning |
|---|---|
| `count` | current state |
| `setCount` | function to update state |
| `useState(0)` | initial value |

---

## 4 How `useState` Works Internally

React stores hooks in an internal linked list/array.

Example:

```jsx
const [count, setCount] = useState(0);
const [name, setName] = useState("React");
```

React internally stores something similar to:

```js
hooks = [0, "React"];
```

React uses the order of hooks to match states during re-render.

That’s why:

❌ Hooks cannot be called conditionally.

---

### Wrong Example

```jsx
if (show) {
  useState(0);
}
```

This breaks hook ordering.

---

## 5 State Update Process

When `setState` is called:

```jsx
setCount(5);
```

React:

1. schedules an update
2. marks component for re-render
3. compares Virtual DOM
4. updates actual DOM efficiently

---

## 6 Functional Updates

Sometimes state depends on previous state.

❌ Wrong:

```jsx
setCount(count + 1);
setCount(count + 1);
```

Expected:
```js
2
```

Actual:
```js
1
```

Because both updates use the same stale value.

---

### ✅ Correct

```jsx
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

Now React correctly increments twice.

---

## 7 Batching in React

React batches multiple updates for performance.

```jsx
setCount(c => c + 1);
setName("React");
setAge(25);
```

React performs:
- one render
- not three renders

---

### Automatic Batching (React 18)

React 18 batches updates inside:
- events
- promises
- timeouts
- async calls

---

## 8 Lazy Initialization

Sometimes initial state is expensive.

❌ Bad

```jsx
const [data] = useState(expensiveFunction());
```

Runs every render.

---

### ✅ Correct

```jsx
const [data] = useState(() => expensiveFunction());
```

Runs only once during initial render.

---

## 9 Object & Array State Updates

`useState` does NOT merge objects automatically.

---

### ❌ Wrong

```jsx
const [user, setUser] = useState({
  name: "John",
  age: 20
});

setUser({
  age: 25
});
```

Result:

```js
{ age: 25 }
```

`name` gets removed.

---

### ✅ Correct

```jsx
setUser(prev => ({
  ...prev,
  age: 25
}));
```

---

## 10 Common Mistakes

---

### 1. Updating State Directly

❌ Wrong

```jsx
count = count + 1;
```

---

### 2. Using State Immediately After Setting

```jsx
setCount(count + 1);

console.log(count);
```

Still logs old value.

Because state updates are asynchronous.

---

### 3. Mutating Arrays

❌ Wrong

```jsx
items.push(newItem);
setItems(items);
```

---

### ✅ Correct

```jsx
setItems(prev => [...prev, newItem]);
```

---

## 11 Important Interview Concepts

---

### State Updates Are Asynchronous

React schedules updates.

```jsx
setCount(1);
console.log(count);
```

Old value is logged.

---

### Re-render Trigger

A component re-renders when:
- state changes
- props change
- parent re-renders

---

### Primitive vs Reference Types

Primitive:
```js
number
string
boolean
```

Reference:
```js
object
array
function
```

Reference types require immutable updates.

---

### Why Hooks Must Be Top Level

React relies on hook order.

Never call hooks:
- inside loops
- inside conditions
- inside nested functions

---

## 12 Real-World Examples

---

### Counter

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <h1>{count}</h1>

      <button onClick={() => setCount(c => c + 1)}>
        Increment
      </button>
    </>
  );
}
```

---

### Toggle

```jsx
function Toggle() {
  const [show, setShow] = useState(false);

  return (
    <button onClick={() => setShow(s => !s)}>
      Toggle
    </button>
  );
}
```

---

### Input Field

```jsx
function InputBox() {
  const [name, setName] = useState("");

  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  );
}
```

---

## 13 Performance Considerations

---

### Avoid Unnecessary State

❌ Bad

```jsx
const [fullName, setFullName] = useState(
  first + last
);
```

Use derived values instead.

---

### Use Lazy Initialization

Good for:
- API parsing
- localStorage
- expensive calculations

---

### Avoid Large Objects

Split state when possible.

❌ Bad

```jsx
const [state, setState] = useState({
  name: "",
  age: "",
  address: "",
  cart: [],
  theme: ""
});
```

---

## 14 Interview Questions

---

### Q1. Why is `useState` asynchronous?

Because React batches updates for performance optimization.

---

### Q2. Why does React not update state immediately?

Immediate updates would cause excessive re-renders.

---

### Q3. Difference between normal and functional update?

Normal update:
```jsx
setCount(count + 1)
```

Functional update:
```jsx
setCount(prev => prev + 1)
```

Functional update is safer when depending on previous state.

---

### Q4. Why hooks cannot be conditional?

Because React tracks hooks by order.

---

### Q5. Does `useState` merge objects?

No.

Unlike class components, `useState` replaces state completely.

---

## 15 Summary

| Topic | Key Point |
|---|---|
| `useState` | manages component state |
| updates | asynchronous |
| batching | improves performance |
| functional updates | prevent stale state |
| objects | must update immutably |
| hooks | must stay top-level |

---

## ✅ Final Notes

For interviews focus deeply on:
- batching
- stale closures
- functional updates
- async behavior
- immutability
- hook ordering
- React rendering cycle

These are frequently asked in:
- React interviews
- machine coding rounds
- senior frontend interviews
