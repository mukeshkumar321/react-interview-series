# React `useState` Tricky Output Questions

> These questions are designed for React interviews and deep understanding of `useState`.

---

## 📚 Table of Contents

1. Basic State Updates
2. Functional Updates
3. Async Behavior
4. Object State
5. Closures
6. Batching
7. Lazy Initialization
8. Re-render Behavior
9. Mutation Problems
10. Advanced Interview Questions

---

## 1 Basic State Updates

### Question 1

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);

    console.log(count);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
0
```

UI becomes:

```js
1
```

</details>

<details>
<summary>👉 Show Explanation</summary>

Both updates use stale value `0`.

React batches updates.

So:

```js
setCount(1)
setCount(1)
```

Final state becomes:

```js
1
```

</details>

---

## 2 Functional Updates

### Question 2

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(c => c + 1);
    setCount(c => c + 1);

    console.log(count);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
0
```

UI becomes:

```js
2
```

</details>

<details>
<summary>👉 Show Explanation</summary>

Functional updates receive latest state.

Flow:

```js
0 -> 1 -> 2
```

</details>

---

## 3 Async Behavior

### Question 3

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    setCount(5);

    console.log(count);
  }, []);

  return <h1>{count}</h1>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
0
```

UI:

```js
5
```

</details>

<details>
<summary>👉 Show Explanation</summary>

State updates are asynchronous.

`console.log` runs before re-render.

</details>

---

## 4 Closure Problem

### Question 4

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setTimeout(() => {
      console.log(count);
    }, 2000);

    setCount(count + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
0
```

</details>

<details>
<summary>👉 Show Explanation</summary>

`setTimeout` captures old closure value.

This is called stale closure problem.

</details>

---

## 5 Object State

### Question 5

```jsx
function App() {
  const [user, setUser] = React.useState({
    name: "John",
    age: 20
  });

  function handleClick() {
    setUser({
      age: 25
    });
  }

  console.log(user);

  return <button onClick={handleClick}>Update</button>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
{
  age: 25
}
```

</details>

<details>
<summary>👉 Show Explanation</summary>

`useState` replaces entire object.

It does NOT merge automatically.

</details>

---

## 6 Mutation Problem

### Question 6

```jsx
function App() {
  const [items, setItems] = React.useState([1, 2]);

  function handleClick() {
    items.push(3);

    setItems(items);
  }

  return <button onClick={handleClick}>{items.length}</button>;
}
```

<details>
<summary>👉 Show Output</summary>

UI may NOT update correctly.

</details>

<details>
<summary>👉 Show Explanation</summary>

State was mutated directly.

React compares references.

Old and new reference are same.

</details>

---

## 7 Lazy Initialization

### Question 7

```jsx
function expensive() {
  console.log("RUN");

  return 10;
}

function App() {
  const [count] = React.useState(expensive());

  return <h1>{count}</h1>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
RUN
```

Runs on every render.

</details>

<details>
<summary>👉 Show Explanation</summary>

Function executes before `useState`.

Use lazy initialization instead.

</details>

---

### Correct Version

```jsx
const [count] = React.useState(() => expensive());
```

Now runs only once.

---

## 8 Batching

### Question 8

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(1);

    setTimeout(() => {
      setCount(2);
      setCount(3);
    });
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

<details>
<summary>👉 Show Output</summary>

Final UI:

```js
3
```

</details>

<details>
<summary>👉 Show Explanation</summary>

React 18 supports automatic batching even inside timeouts.

</details>

---

## 9 Infinite Re-render

### Question 9

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  setCount(1);

  return <h1>{count}</h1>;
}
```

<details>
<summary>👉 Show Output</summary>

Infinite re-render error.

</details>

<details>
<summary>👉 Show Explanation</summary>

State update happens during render.

Causes endless render loop.

</details>

---

## 10 Reference Equality

### Question 10

```jsx
function App() {
  const [user, setUser] = React.useState({
    name: "React"
  });

  function handleClick() {
    setUser(user);
  }

  return <button onClick={handleClick}>Click</button>;
}
```

<details>
<summary>👉 Show Output</summary>

No re-render.

</details>

<details>
<summary>👉 Show Explanation</summary>

Same object reference.

React skips re-render.

</details>

---

## 11 Functional Update Queue

### Question 11

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(c => c + 1);

    setCount(c => c * 2);

    setCount(c => c + 5);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

<details>
<summary>👉 Show Output</summary>

```js
7
```

</details>

<details>
<summary>👉 Show Explanation</summary>

Flow:

```js
0 + 1 = 1
1 * 2 = 2
2 + 5 = 7
```

</details>

---

## 12 State Preservation

### Question 12

```jsx
function Child() {
  const [count] = React.useState(0);

  console.log("Child Render");

  return <h1>{count}</h1>;
}

function App() {
  const [show, setShow] = React.useState(true);

  return (
    <>
      <button onClick={() => setShow(!show)}>
        Toggle
      </button>

      {show && <Child />}
    </>
  );
}
```

<details>
<summary>👉 Show Output</summary>

Child state resets after remount.

</details>

<details>
<summary>👉 Show Explanation</summary>

Unmounting destroys component state.

Remount creates fresh state.

</details>

---

## ✅ Interview Tips

Focus deeply on:
- stale closures
- batching
- async state updates
- immutability
- reference equality
- render cycle
- functional updates

These concepts are heavily asked in:
- React interviews
- frontend interviews
- machine coding rounds
- senior React roles
