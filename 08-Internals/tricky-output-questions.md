## 📚 React Internals — Tricky Output Questions

> Questions focusing on reconciliation, fiber architecture, batching, rendering order, and internal mechanics.

---

## 1. Reconciliation & Diffing

### Q1

```jsx
function App() {
  const [flag, setFlag] = useState(true);

  return (
    <div>
      {flag ? <Input /> : <p>Empty</p>}
      <button onClick={() => setFlag(!flag)}>Toggle</button>
    </div>
  );
}

function Input() {
  const [value, setValue] = useState("initial");
  
  useEffect(() => {
    console.log("Input mounted");
  }, []);

  return <input value={value} onChange={e => setValue(e.target.value)} />;
}
```

#### ❓ What happens when Toggle is clicked?

<details>
<summary>✅ Answer</summary>

When flag becomes false:

1. `<Input />` changes to `<p>Empty</p>`
2. Input component instance is destroyed
3. Input state is lost
4. When toggled back, a NEW Input instance is created
5. "Input mounted" logs again

**Explanation:** Different element types cause React to destroy the old fiber tree entirely. The Input component is unmounted, losing all state. It's a completely new instance when toggled back.

</details>

---

### Q2

```jsx
function App() {
  const [items, setItems] = useState(["a", "b", "c"]);

  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
      <button onClick={() => setItems(["x", ...items.slice(1)])}>
        Change First
      </button>
    </ul>
  );
}
```

#### ❓ What renders after clicking button?

<details>
<summary>✅ Answer</summary>

```
- x
- b
- c
```

But there's a problem: index keys are unreliable. If the first item had state, it would be lost.

**Explanation:** Using index as key works for output but is problematic. React matches `key={0}` to the first item regardless of content. When content changes, state/focus mismatches occur.

</details>

---

### Q3

```jsx
function App() {
  const [items, setItems] = useState([
    { id: 1, text: "a" },
    { id: 2, text: "b" },
    { id: 3, text: "c" }
  ]);

  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.text}</li>
      ))}
      <button onClick={() => setItems([
        { id: 10, text: "x" }, ...items.slice(1)
      ])}>
        Change First
      </button>
    </ul>
  );
}
```

#### ❓ What renders after clicking button?

<details>
<summary>✅ Answer</summary>

```
- x
- b
- c
```

With proper key matching:
- Item id=1 is removed (no longer in list)
- Item id=10 is new
- Items id=2 and id=3 remain and reuse their DOM nodes

React correctly matches items by id and updates only what changed.

</details>

---

## 2. Fiber & Rendering Order

### Q4

```jsx
function App() {
  const [count, setCount] = useState(0);

  console.log("App render");

  return (
    <div>
      <h1>{count}</h1>
      <Child count={count} />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

function Child({ count }) {
  console.log("Child render");
  
  useEffect(() => {
    console.log("Child effect");
  }, [count]);

  return <p>{count}</p>;
}
```

#### ❓ Console output when button is clicked?

<details>
<summary>✅ Answer</summary>

```
App render
Child render
Child effect
```

**Explanation:**

1. **Render Phase (synchronous):**
   - App renders → "App render"
   - Child renders → "Child render"

2. **Commit Phase:**
   - Effects run → "Child effect"

Renders happen for all components first (depth-first), then effects run.

</details>

---

### Q5

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("App effect");
  }, [count]);

  console.log("App render");

  return (
    <div>
      <h1>{count}</h1>
      <Child count={count} />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

function Child({ count }) {
  useEffect(() => {
    console.log("Child effect");
  }, [count]);

  console.log("Child render");

  return <p>{count}</p>;
}
```

#### ❓ Console output when button is clicked?

<details>
<summary>✅ Answer</summary>

```
App render
Child render
App effect
Child effect
```

**Explanation:**

1. **Render Phase:**
   - App → "App render"
   - Child → "Child render"

2. **Commit Phase (effects run in order):**
   - App effect → "App effect"
   - Child effect → "Child effect"

Effects run in the same order as components rendered.

</details>

---

## 3. Batching

### Q6

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = async () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  };

  return <button onClick={handleClick}>Count: {count}</button>;
}
```

#### ❓ Count after clicking button?

<details>
<summary>✅ Answer</summary>

```
1
```

All three calls are in the same event callback, so React batches them. All use the same count value (0), so all evaluate to `0 + 1 = 1`.

</details>

---

### Q7

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    
    setTimeout(() => {
      setCount(c => c + 1);
      setCount(c => c + 1);
    }, 0);
  };

  return <button onClick={handleClick}>Count: {count}</button>;
}
```

#### ❓ Count after clicking button?

<details>
<summary>✅ Answer</summary>

In React 18:
```
3
```

In React 17:
```
2
```

**Explanation:**

- First `setCount(count + 1)` updates to 1
- setTimeout callback: React 18 batches the two updates
- Functional updates chain: 1→2, then 2→3

React 18 batches setTimeout automatically. React 17 doesn't, causing separate updates.

</details>

---

## 4. Component Lifecycle

### Q8

```jsx
function App() {
  const [show, setShow] = useState(true);

  return (
    <div>
      {show && <StrictModeChild />}
      <button onClick={() => setShow(!show)}>Toggle</button>
    </div>
  );
}

function StrictModeChild() {
  useEffect(() => {
    console.log("Mount");
    return () => console.log("Unmount");
  }, []);

  return <div>Child</div>;
}

export default function Root() {
  return (
    <React.StrictMode>
      <App />
    </React.StrictMode>
  );
}
```

#### ❓ Console output when toggling?

<details>
<summary>✅ Answer</summary>

First toggle (hide):
```
Mount
Mount
Unmount
Unmount
```

Second toggle (show):
```
Mount
Mount
Unmount
Unmount
```

**Explanation:** React.StrictMode intentionally double-invokes effects during development to detect side effects. This helps identify components that don't properly clean up.

</details>

---

### Q9

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("Parent effect");
    return () => console.log("Parent cleanup");
  }, []);

  return (
    <div>
      <h1>{count}</h1>
      <Child />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

function Child() {
  useEffect(() => {
    console.log("Child effect");
    return () => console.log("Child cleanup");
  }, []);

  return <p>Child</p>;
}
```

#### ❓ Console output on initial mount?

<details>
<summary>✅ Answer</summary>

```
Parent effect
Child effect
```

**Explanation:** Effects run after render phase completes. Components render depth-first (Parent → Child), and effects run in the same order. On unmount, cleanup functions run in reverse order (Child cleanup, then Parent cleanup).

</details>

---

## 5. Keys & Re-mounting

### Q10

```jsx
function App() {
  const [items, setItems] = useState([1, 2, 3]);
  const [key, setKey] = useState(0);

  return (
    <div key={key}>
      {items.map(item => (
        <Item key={item} id={item} />
      ))}
      <button onClick={() => setKey(k => k + 1)}>
        Reset Key
      </button>
    </div>
  );
}

function Item({ id }) {
  const [value, setValue] = useState(id);

  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
}
```

#### ❓ What happens when "Reset Key" is clicked?

<details>
<summary>✅ Answer</summary>

1. The div's key changes
2. Entire div subtree is destroyed and recreated
3. All Item components are unmounted and remounted
4. All input values reset to their initial values (1, 2, 3)
5. Any typed values are lost

**Explanation:** Changing a parent element's key destroys and recreates its entire subtree. This forces a complete re-mount of all children.

</details>

---

## 6. Closures & Stale State

### Q11

```jsx
function App() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setTimeout(() => {
      console.log(count);
      setCount(count + 1);
    }, 3000);
  };

  return (
    <div>
      <h1>{count}</h1>
      <button onClick={handleClick}>Click</button>
    </div>
  );
}
```

#### ❓ Click at count=0, then at count=1, what logs?

<details>
<summary>✅ Answer</summary>

Assuming 3 seconds between clicks:

```
0  (first timeout)
1  (second timeout)
```

**Explanation:** Each click creates a timeout that closes over the `count` value at that moment. The callback has access to the count from when that click happened, not the current count. This is a closure behavior.

</details>

---

### Q12

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1);
    }, 1000);

    return () => clearInterval(interval);
  }, [count]);

  return <h1>{count}</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

```
0 → 1 → 2 → 3 → ...
```

Increments every second indefinitely.

**Explanation:** The dependency array includes `count`. Every time count changes:
1. Cleanup clears the old interval
2. Effect runs with new count
3. New interval is created

This works but creates/destroys intervals frequently.

Better: Use functional update and remove count from dependencies.

</details>

---

## 7. Performance & Memoization

### Q13

```jsx
function App() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
  };

  return (
    <>
      <Child onClick={handleClick} />
      <h1>{count}</h1>
    </>
  );
}

function Child({ onClick }) {
  console.log("Child rendered");
  return <button onClick={onClick}>Click</button>;
}
```

#### ❓ When App button is clicked, does Child re-render?

<details>
<summary>✅ Answer</summary>

```
Child rendered
```

Yes, Child re-renders. When App's state changes, App re-renders. The `onClick` prop becomes a new function reference, so Child receives a new prop and re-renders.

Solution: Use `React.memo` or `useCallback`.

</details>

---

### Q14

```jsx
const Child = React.memo(({ count }) => {
  console.log("Child rendered", count);
  return <p>{count}</p>;
});

function App() {
  const [count, setCount] = useState(0);

  return (
    <>
      <Child count={count} />
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </>
  );
}
```

#### ❓ When button is clicked, does Child re-render?

<details>
<summary>✅ Answer</summary>

```
Child rendered 1
```

Yes, Child re-renders because the `count` prop changed. `React.memo` only prevents re-renders when props DON'T change.

</details>

---

## 8. Advanced Scenarios

### Q15

```jsx
function App() {
  const [condition, setCondition] = useState(true);

  if (condition) {
    const [count, setCount] = useState(0);
    return <div>{count}</div>;
  }

  return <div>Else</div>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

React throws an error:

```
React Hook "useState" is called conditionally.
React Hooks must be called in the same order in every component render.
```

**Explanation:** Hooks must be at the top level of the component, not inside conditionals. React relies on hook call order to match hooks to state.

</details>

---

### Q16

```jsx
function App() {
  const [count, setCount] = useState(0);

  const items = [1, 2, 3];
  items.forEach(() => {
    const [value, setValue] = useState(0);
  });

  return <h1>{count}</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

React throws an error:

```
React Hook "useState" cannot be called inside a callback.
React Hooks must be called in the same order in every component render.
```

**Explanation:** Hooks can't be called inside loops. The call count would vary if the array length changes, breaking React's hook ordering system.

</details>

---

### Q17

```jsx
function App() {
  const ref = useRef(null);
  const [count, setCount] = useState(0);

  useEffect(() => {
    if (ref.current) {
      ref.current.textContent = count;
    }
  }, [count]);

  return (
    <div>
      <div ref={ref}></div>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

#### ❓ What displays after clicking button?

<details>
<summary>✅ Answer</summary>

```
1
2
3
...
```

The effect runs after each render, updating the div's text with the current count. Refs persist across renders without triggering re-renders.

</details>

---

### Q18

```jsx
function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <h1 key={count}>{count}</h1>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

#### ❓ What happens when button is clicked?

<details>
<summary>✅ Answer</summary>

1. New h1 element is created with key={1}
2. Old h1 element with key={0} is destroyed
3. The new h1 mounts
4. Any internal state in the h1 is lost

**Explanation:** Changing an element's key destroys and recreates it. Useful for resetting component state, but expensive for performance.

</details>
