# React Hooks — Tricky Output Questions ⚛️

These questions focus on:

- Hooks Internals
- Closures
- Render Cycles
- Referential Equality
- React.memo
- useMemo
- useCallback
- Hook Rules
- Fiber Behavior
- Render vs Commit Phase

---

## 📚 Table of Contents

- [1. Introduction to Hooks Questions](#1-introduction-to-hooks-questions)
- [2. Rules of Hooks Questions](#2-rules-of-hooks-questions)
- [3. Hook Internals Questions](#3-hook-internals-questions)
- [4. Performance Patterns Questions](#4-performance-patterns-questions)

---

## 1. Introduction to Hooks Questions

### Question 1 — Closure Snapshot

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function log() {
    console.log(count);
  }

  React.useEffect(() => {
    setTimeout(log, 1000);
  }, []);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

### ❓ What will be logged after clicking the button multiple times?

<details>
<summary>✅Output</summary>

```txt
0
```


`log` captures the closure from the initial render.

The effect runs only once because of `[]`.

So `setTimeout` always uses the old closure where:

```js
count = 0
```

This is called a stale closure.

</details>

---

### Question 2 — Render Cycle

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  console.log("render");

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

### ❓ What happens after clicking the button 3 times?

<details>
<summary>✅Output</summary>

```txt
render
render
render
render
```


- Initial render → 1 render
- 3 clicks → 3 additional renders

Total:

```txt
4 renders
```

Every state update triggers re-render.

</details>

---

### Question 3 — New Function Per Render

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const fn1 = () => {};

  const fn2 = () => {};

  console.log(fn1 === fn2);

  return (
    <button onClick={() => setCount(count + 1)}>
      Click
    </button>
  );
}
```

### ❓ What will be logged?

<details>
<summary>✅Output</summary>

```txt
false
false
false
...
```


Every render creates NEW function references.

Functions compare by reference.

</details>

---

## 2. Rules of Hooks Questions

### Question 4 — Conditional Hook

```jsx
function App() {
  const [show, setShow] = React.useState(true);

  if (show) {
    React.useEffect(() => {
      console.log("effect");
    }, []);
  }

  return (
    <button onClick={() => setShow(false)}>
      Toggle
    </button>
  );
}
```

### ❓ What happens after clicking the button?

<details>
<summary>✅Output</summary>

```txt
React Hook Error
```


Hooks must execute in the same order every render.

First render:

```txt
0 → useState
1 → useEffect
```

Second render:

```txt
0 → useState
```

Hook order breaks.

</details>

---

### Question 5 — Hook Indexing

```jsx
function App() {
  const [a] = React.useState("A");

  const [b] = React.useState("B");

  console.log(a, b);

  return null;
}
```

### ❓ How does React internally track these hooks?

<details>
<summary>✅ Internal Representation</summary>

```txt
hooks[0] → "A"
hooks[1] → "B"
```

</details>

<details>


React tracks hooks by execution order, NOT variable names.

</details>

---

### Question 6 — Hook Inside Loop

```jsx
function App() {
  for (let i = 0; i < 3; i++) {
    React.useState(0);
  }

  return null;
}
```

### ❓ What happens?

<details>
<summary>✅Output</summary>

```txt
React Hook Error
```


Hooks cannot run inside loops because execution order can change.

</details>

---

## 3. Hook Internals Questions

### Question 7 — State Queue

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);

    console.log(count);
  }

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

### ❓ What happens after clicking?

<details>
<summary>✅Output</summary>

```txt
0
```

UI becomes:

```txt
1
```


Both updates use the SAME closure value:

```js
count = 0
```

React batches updates.

Both become:

```js
setCount(1);
setCount(1);
```

Final state:

```txt
1
```

</details>

---

### Question 8 — Functional Updates

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(c => c + 1);
    setCount(c => c + 1);
  }

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

### ❓ What will UI show after clicking once?

<details>
<summary>✅Output</summary>

```txt
2
```


Functional updates always receive latest state.

Sequence:

```txt
0 → 1 → 2
```

</details>

---

### Question 9 — Render vs Commit Phase

```jsx
function App() {
  console.log("render");

  React.useEffect(() => {
    console.log("effect");
  });

  return <h1>Hello</h1>;
}
```

### ❓ What is the output order?

<details>
<summary>✅Output</summary>

```txt
render
effect
```


`useEffect` runs AFTER commit phase.

Sequence:

```txt
Render Phase
↓
Commit Phase
↓
Effect Runs
```

</details>

---

### Question 10 — Cleanup Timing

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    console.log("effect", count);

    return () => {
      console.log("cleanup", count);
    };
  }, [count]);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

### ❓ What happens after clicking once?

<details>
<summary>✅Output</summary>

```txt
effect 0
cleanup 0
effect 1
```


Cleanup runs BEFORE next effect execution.

</details>

---

### Question 11 — Closure Snapshot

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function log() {
    setTimeout(() => {
      console.log(count);
    }, 1000);
  }

  return (
    <>
      <button onClick={log}>Log</button>

      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </>
  );
}
```

### ❓ Steps

1. Click "Log"
2. Click "Increment"

<details>
<summary>✅Output</summary>

```txt
0
```


The timeout callback captures the closure snapshot from when `log` was clicked.

</details>

---

## 4. Performance Patterns Questions

### Question 12 — React.memo Failure

```jsx
const Child = React.memo(({ user }) => {
  console.log("child render");

  return <h1>{user.name}</h1>;
});

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <>
      <Child user={{ name: "John" }} />

      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>
    </>
  );
}
```

### ❓ Does child re-render?

<details>
<summary>✅Output</summary>

```txt
child render
child render
child render
...
```


New object reference created every render:

```js
{ name: "John" }
```

`React.memo` uses shallow comparison.

</details>

---

### Question 13 — useMemo Fix

```jsx
const Child = React.memo(({ user }) => {
  console.log("child render");

  return <h1>{user.name}</h1>;
});

function App() {
  const [count, setCount] = React.useState(0);

  const user = React.useMemo(() => {
    return { name: "John" };
  }, []);

  return (
    <>
      <Child user={user} />

      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>
    </>
  );
}
```

### ❓ Does child re-render now?

<details>
<summary>✅Output</summary>

```txt
child render
```


`useMemo` preserves object reference.

</details>

---

### Question 14 — useCallback Reference

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const fn = React.useCallback(() => {}, []);

  console.log(fn);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

### ❓ Does `fn` change across renders?

<details>
<summary>✅Output</summary>

```txt
Same function reference
```


`useCallback` memoizes function reference.

</details>

---

### Question 15 — Referential Equality

```jsx
function App() {
  const arr1 = [];

  const arr2 = [];

  console.log(arr1 === arr2);

  return null;
}
```

### ❓ What will be logged?

<details>
<summary>✅Output</summary>

```txt
false
```


Arrays compare by reference.

Each render creates new array instances.

</details>

---

### Question 16 — useEffect Dependency Trap

```jsx
function App() {
  const obj = {};

  React.useEffect(() => {
    console.log("effect");
  }, [obj]);

  return <h1>Hello</h1>;
}
```

### ❓ What happens on every render?

<details>
<summary>✅Output</summary>

```txt
effect
effect
effect
...
```


`obj` creates new reference every render.

Dependency comparison fails every time.

</details>

---

### Question 17 — Context Re-render

```jsx
const ThemeContext = React.createContext();

function Child() {
  console.log("child render");

  return <h1>Child</h1>;
}

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <ThemeContext.Provider value={{ dark: true }}>
      <Child />

      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>
    </ThemeContext.Provider>
  );
}
```

### ❓ Does Child re-render?

<details>
<summary>✅Output</summary>

```txt
child render
child render
child render
...
```


Provider value creates new object reference every render.

All consumers re-render.

</details>

---

### Question 18 — Virtualization Concept

```jsx
function App() {
  return items.map(item => {
    return <Row key={item.id} />;
  });
}
```

### ❓ Why can this become slow for 50,000 items?

<details>
<summary>✅ Explanation</summary>

React creates thousands of DOM nodes.

Large DOM trees are expensive.

Virtualization solves this by rendering only visible items.

Libraries:

- react-window
- react-virtualized

</details>

---

## Summary

These questions test deep understanding of:

- Closures
- Render Cycles
- Hook Internals
- Referential Equality
- React.memo
- useMemo
- useCallback
- Effect Timing
- Hook Rules
- Fiber Concepts

These are extremely common in:

- React Interviews
- Senior Frontend Interviews
- Machine Coding Rounds
- React Debugging Scenarios
- Performance Optimization Discussions