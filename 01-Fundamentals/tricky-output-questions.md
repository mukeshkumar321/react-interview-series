## 📚 React Fundamentals — Tricky Output Questions

> These questions are focused on real React behavior, rendering logic, hooks, JSX, reconciliation, state updates, closures, batching, and edge cases.

---

## 1. JSX & Rendering

### Q1

```jsx
function App() {
  const name = "React";

  return <h1>Hello {name}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
Hello React
```

</details>

---

### Q2

```jsx
function App() {
  return <h1>{true}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

Nothing renders.

React ignores:
- `true`
- `false`
- `null`
- `undefined`

</details>

---

### Q3

```jsx
function App() {
  return <h1>{0}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
0
```

`0` is rendered because it is a valid value.

</details>

---

### Q4

```jsx
function App() {
  const user = {
    name: "John",
  };

  return <h1>{user}</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

React throws an error:

```txt
Objects are not valid as a React child
```

</details>

---

### Q5

```jsx
function App() {
  return <h1>{["A", "B", "C"]}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
ABC
```

Arrays are rendered.

</details>

---

## 2. State Updates

### Q6

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ After one click?

<details>
<summary>✅ Answer</summary>

```txt
1
```

Because both updates use the same stale value.

</details>

---

### Q7

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ After one click?

<details>
<summary>✅ Answer</summary>

```txt
2
```

Functional updates use latest state.

</details>

---

### Q8

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  console.log("Render");

  return (
    <button onClick={() => setCount(count)}>
      {count}
    </button>
  );
}
```

#### ❓ Will component re-render?

<details>
<summary>✅ Answer</summary>

No.

React skips re-render if state value is unchanged.

</details>

---

## 3. Closures in React

### Q9

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function log() {
    setTimeout(() => {
      console.log(count);
    }, 2000);
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>

      <button onClick={log}>
        Log
      </button>
    </>
  );
}
```

#### ❓ What gets logged?

<details>
<summary>✅ Answer</summary>

Closure captures old state value.

It logs the value from the render when `log()` was called.

</details>

---

## 4. Event Handling

### Q10

```jsx
function App() {
  function handleClick() {
    console.log("Clicked");
  }

  return <button onClick={handleClick()}>Click</button>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

Function executes immediately during render.

Correct:

```jsx
onClick={handleClick}
```

</details>

---

### Q11

```jsx
<button onClick={() => console.log("Hi")}>
  Click
</button>
```

#### ❓ When does console run?

<details>
<summary>✅ Answer</summary>

Only after click.

Arrow function delays execution.

</details>

---

## 5. Conditional Rendering

### Q12

```jsx
function App() {
  return <div>{false && <h1>Hello</h1>}</div>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

Nothing renders.

</details>

---

### Q13

```jsx
function App() {
  return <div>{0 && <h1>Hello</h1>}</div>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
0
```

Because `0` is rendered.

</details>

---

## 6. Lists & Keys

### Q14

```jsx
const users = ["A", "B", "C"];

function App() {
  return (
    <>
      {users.map(user => (
        <h1>{user}</h1>
      ))}
    </>
  );
}
```

#### ❓ What warning appears?

<details>
<summary>✅ Answer</summary>

```txt
Each child in a list should have a unique "key" prop.
```

</details>

---

### Q15

```jsx
{
  users.map((user, index) => (
    <h1 key={index}>{user}</h1>
  ));
}
```

#### ❓ Why can index keys be problematic?

<details>
<summary>✅ Answer</summary>

They can cause incorrect UI updates during:
- insertions
- deletions
- reordering

</details>

---

## 7. useEffect

### Q16

```jsx
function App() {
  React.useEffect(() => {
    console.log("Effect");
  });

  return <h1>Hello</h1>;
}
```

#### ❓ When does effect run?

<details>
<summary>✅ Answer</summary>

After every render.

</details>

---

### Q17

```jsx
React.useEffect(() => {
  console.log("Mounted");
}, []);
```

#### ❓ When does this run?

<details>
<summary>✅ Answer</summary>

Only once after initial mount.

</details>

---

### Q18

```jsx
React.useEffect(() => {
  console.log("Updated");
}, [count]);
```

#### ❓ When does this run?

<details>
<summary>✅ Answer</summary>

- Initial render
- Whenever `count` changes

</details>

---

## 8. Infinite Re-render

### Q19

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  setCount(1);

  return <h1>{count}</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

Infinite re-render occurs.

Because state update happens during render.

</details>

---

## 9. Controlled Components

### Q20

```jsx
function App() {
  const [value, setValue] = React.useState("");

  return (
    <input
      value={value}
    />
  );
}
```

#### ❓ Can user type?

<details>
<summary>✅ Answer</summary>

No.

Because input has no `onChange`.

</details>

---

## 10. Reconciliation


### Q21

```jsx
function App() {
  return (
    <>
      <h1>Hello</h1>
      <h1>Hello</h1>
    </>
  );
}
```

#### ❓ Does React create full DOM every render?

<details>
<summary>✅ Answer</summary>

No.

React uses Virtual DOM diffing and reconciliation.

</details>

---

## 11. Strict Mode

### Q22

```jsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

#### ❓ Why might component render twice in development?

<details>
<summary>✅ Answer</summary>

Strict Mode intentionally double-invokes components in development to detect side effects.

</details>

---

## 12. React.memo

### Q23

```jsx
const Child = React.memo(() => {
  console.log("Child Render");

  return <h1>Child</h1>;
});
```

#### ❓ When does child re-render?

<details>
<summary>✅ Answer</summary>

Only when props change.

</details>

---

## 13. useRef

### Q24

```jsx
function App() {
  const ref = React.useRef(0);

  function update() {
    ref.current++;
    console.log(ref.current);
  }

  return <button onClick={update}>Click</button>;
}
```

#### ❓ Does component re-render?

<details>
<summary>✅ Answer</summary>

No.

`useRef` changes do not trigger re-render.

</details>

---

## 14. Batching

### Q25

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

#### ❓ What gets logged?

<details>
<summary>✅ Answer</summary>

```txt
0
```

Because state updates are asynchronous and batched.

</details>

---

## 15. Fragment

### Q26

```jsx
function App() {
  return (
    <>
      <h1>Hello</h1>
      <h2>World</h2>
    </>
  );
}
```

#### ❓ Why use Fragment?

<details>
<summary>✅ Answer</summary>

To avoid unnecessary DOM nodes.

</details>

---

## 16. Component Rendering

### Q27

```jsx
function Child() {
  console.log("Child");

  return <h1>Child</h1>;
}

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <>
      <Child />
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>
    </>
  );
}
```

#### ❓ Does Child re-render?

<details>
<summary>✅ Answer</summary>

Yes.

Parent re-render causes child re-render.

</details>

---

## 17. Derived State

### Q28

```jsx
function App() {
  const [count] = React.useState(5);

  const doubled = count * 2;

  return <h1>{doubled}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
10
```

</details>

---

## 18. Boolean Rendering Edge Cases-

### Q29

```jsx
function App() {
  return <h1>{NaN}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
NaN
```

</details>

---

### Q30

```jsx
function App() {
  return <h1>{undefined}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

Nothing renders.

</details>

---

## ✅ Topics Covered

- JSX
- Rendering
- State
- Batching
- Closures
- Event Handling
- Conditional Rendering
- Lists & Keys
- useEffect
- Infinite Re-renders
- Controlled Components
- Reconciliation
- Strict Mode
- React.memo
- useRef
- Fragment
- Parent/Child Rendering
- Edge Cases
