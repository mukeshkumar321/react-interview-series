# React Internals Tricky Output Questions

# Table of Contents

1. [Render Cycle Questions](#1-render-cycle-questions)
2. [Batching Questions](#2-batching-questions)
3. [Closure Questions](#3-closure-questions)
4. [useEffect Questions](#4-useeffect-questions)
5. [Reconciliation Questions](#5-reconciliation-questions)
6. [Key Prop Questions](#6-key-prop-questions)
7. [React.memo Questions](#7-reactmemo-questions)
8. [useMemo & useCallback Questions](#8-usememo--usecallback-questions)
9. [StrictMode Questions](#9-strictmode-questions)
10. [Concurrent Rendering Concepts](#10-concurrent-rendering-concepts)

---

# 1. Render Cycle Questions

---

## Question 1

```jsx
function Child() {
  console.log("Child Render");

  return <h1>Child</h1>;
}

export default function App() {
  const [count, setCount] = React.useState(0);

  console.log("App Render");

  return (
    <>
      <Child />

      <button
        onClick={() => setCount(count + 1)}
      >
        Increment
      </button>
    </>
  );
}
```

### Output?

<details>
<summary>Answer</summary>

```txt
Initial Render

App Render
Child Render

After Click

App Render
Child Render
```

### Explanation

Parent re-render causes child re-render.

React re-executes component functions.

</details>

---

## Question 2

```jsx
function Child() {
  console.log("Child Render");

  return <h1>Child</h1>;
}

const MemoChild = React.memo(Child);

export default function App() {
  const [count, setCount] = React.useState(0);

  console.log("App Render");

  return (
    <>
      <MemoChild />

      <button
        onClick={() => setCount(count + 1)}
      >
        Increment
      </button>
    </>
  );
}
```

### Output?

<details>
<summary>Answer</summary>

```txt
Initial Render

App Render
Child Render

After Click

App Render
```

### Explanation

`React.memo` skips child re-render because props unchanged.

</details>

---

# 2. Batching Questions

---

## Question 3

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
  };

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

### Final State?

<details>
<summary>Answer</summary>

```txt
1
```

### Explanation

Both updates capture same stale value.

Equivalent to:

```js
setCount(1);
setCount(1);
```

</details>

---

## Question 4

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
  };

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

### Final State?

<details>
<summary>Answer</summary>

```txt
2
```

### Explanation

Functional updates use latest queue value.

Flow:

```txt
0 → 1 → 2
```

</details>

---

## Question 5

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  console.log("Render", count);

  return (
    <button
      onClick={() => {
        setCount(c => c + 1);
        setCount(c => c + 1);

        console.log(count);
      }}
    >
      Click
    </button>
  );
}
```

### Output After Click?

<details>
<summary>Answer</summary>

```txt
0
Render 2
```

### Explanation

State updates are asynchronous.

Inside same event:
- old state still visible

</details>

---

# 3. Closure Questions

---

## Question 6

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setTimeout(() => {
      console.log(count);
    }, 1000);
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>

      <button onClick={handleClick}>
        Log
      </button>
    </>
  );
}
```

### What Gets Logged?

<details>
<summary>Answer</summary>

```txt
Stale value at time of timeout creation
```

### Explanation

Closure captures old state.

</details>

---

## Question 7

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    console.log(count);
  }, []);

  return (
    <button
      onClick={() => setCount(count + 1)}
    >
      {count}
    </button>
  );
}
```

### Output?

<details>
<summary>Answer</summary>

```txt
0
```

### Explanation

Effect runs once because dependency array is empty.

</details>

---

# 4. useEffect Questions

---

## Question 8

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    console.log("Effect");

    return () => {
      console.log("Cleanup");
    };
  }, [count]);

  return (
    <button
      onClick={() => setCount(count + 1)}
    >
      {count}
    </button>
  );
}
```

### Output After Click?

<details>
<summary>Answer</summary>

```txt
Cleanup
Effect
```

### Explanation

Cleanup runs before next effect execution.

Flow:

```txt
Old Cleanup
 ↓
New Effect
```

</details>

---

## Question 9

```jsx
export default function App() {
  console.log("Render");

  React.useEffect(() => {
    console.log("Effect");
  });

  return <h1>Hello</h1>;
}
```

### Output Order?

<details>
<summary>Answer</summary>

```txt
Render
Effect
```

### Explanation

Effects run after commit phase.

</details>

---

# 5. Reconciliation Questions

---

## Question 10

```jsx
export default function App() {
  const [toggle, setToggle] = React.useState(false);

  return (
    <>
      {toggle ? <h1>Hello</h1> : <div>Hello</div>}

      <button onClick={() => setToggle(!toggle)}>
        Toggle
      </button>
    </>
  );
}
```

### What Happens Internally?

<details>
<summary>Answer</summary>

```txt
Entire subtree replaced
```

### Explanation

Different element types:
- `h1`
- `div`

React destroys old subtree.

</details>

---

## Question 11

```jsx
function Input() {
  const [value, setValue] = React.useState("");

  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
}

export default function App() {
  const [toggle, setToggle] = React.useState(false);

  return (
    <>
      {toggle ? <Input /> : <Input />}

      <button onClick={() => setToggle(!toggle)}>
        Toggle
      </button>
    </>
  );
}
```

### Will Input State Reset?

<details>
<summary>Answer</summary>

```txt
No
```

### Explanation

Same component type at same position.

React preserves state.

</details>

---

# 6. Key Prop Questions

---

## Question 12

```jsx
{
  items.map((item, index) => (
    <Input key={index} />
  ));
}
```

### Why Is This Dangerous?

<details>
<summary>Answer</summary>

```txt
State can move to wrong items
```

### Explanation

Index keys break reconciliation when:
- items reordered
- items inserted
- items removed

</details>

---

## Question 13

```jsx
{
  items.map(item => (
    <Input key={Math.random()} />
  ));
}
```

### What Happens?

<details>
<summary>Answer</summary>

```txt
All components remount every render
```

### Explanation

Keys change every render.

React treats every item as new.

</details>

---

# 7. React.memo Questions

---

## Question 14

```jsx
const Child = React.memo(({ obj }) => {
  console.log("Child");

  return <h1>Child</h1>;
});

export default function App() {
  const [count, setCount] = React.useState(0);

  return (
    <>
      <Child obj={{ name: "React" }} />

      <button
        onClick={() => setCount(count + 1)}
      >
        Click
      </button>
    </>
  );
}
```

### Will Child Re-render?

<details>
<summary>Answer</summary>

```txt
Yes
```

### Explanation

New object reference created every render.

Shallow comparison fails.

</details>

---

# 8. useMemo & useCallback Questions

---

## Question 15

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const data = React.useMemo(() => {
    return { name: "React" };
  }, []);

  console.log(data);

  return (
    <button
      onClick={() => setCount(count + 1)}
    >
      Click
    </button>
  );
}
```

### Does Object Reference Change?

<details>
<summary>Answer</summary>

```txt
No
```

### Explanation

`useMemo` caches value.

Dependencies unchanged.

</details>

---

## Question 16

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const fn = React.useCallback(() => {
    console.log(count);
  }, []);

  return (
    <button onClick={fn}>
      Click
    </button>
  );
}
```

### What Gets Logged After Multiple Updates?

<details>
<summary>Answer</summary>

```txt
0
```

### Explanation

Callback captured stale closure.

Dependency array empty.

</details>

---

# 9. StrictMode Questions

---

## Question 17

```jsx
export default function App() {
  React.useEffect(() => {
    console.log("Effect");
  }, []);

  return <h1>Hello</h1>;
}
```

### Why Does Effect Run Twice In Development?

<details>
<summary>Answer</summary>

```txt
React StrictMode
```

### Explanation

StrictMode intentionally double invokes effects in development.

Helps detect side effects.

Production unaffected.

</details>

---

## Question 18

```jsx
export default function App() {
  console.log("Render");

  return <h1>Hello</h1>;
}
```

### Why Can Render Run Twice In Development?

<details>
<summary>Answer</summary>

```txt
StrictMode double rendering
```

### Explanation

React checks render purity.

</details>

---

# 10. Concurrent Rendering Concepts

---

## Question 19

```jsx
startTransition(() => {
  setSearchResults(data);
});

setInputValue(value);
```

### Which Update Gets Priority?

<details>
<summary>Answer</summary>

```txt
setInputValue
```

### Explanation

Transition updates are lower priority.

Urgent UI updates processed first.

</details>

---

## Question 20

```jsx
function App() {
  while (true) {}

  return <h1>Hello</h1>;
}
```

### Can Concurrent Rendering Save This?

<details>
<summary>Answer</summary>

```txt
No
```

### Explanation

JavaScript is still single-threaded.

Infinite loops block main thread completely.

</details>

---

# Important Internal Concepts Tested

These questions test:

- render cycle
- batching
- stale closures
- reconciliation
- hook internals
- memoization
- StrictMode
- concurrent rendering
- effect timing
- state queues

---

# Final Mental Model

To master React Internals, always think:

```txt
State Update
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
Effects
 ↓
Browser Paint
```