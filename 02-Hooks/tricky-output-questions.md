## 📚 React Hooks — Tricky Output Questions

> These questions are focused on hook behavior, rendering, closures, state updates, and edge cases.

---

## 1. Rules of Hooks

### Q1

```jsx
function App({ isAdmin }) {
  if (isAdmin) {
    const [role, setRole] = useState("admin");
  }

  return <h1>App</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

React throws an error:

```txt
React Hook "useState" is called conditionally.
React Hooks must be called in the same order in every component render.
```

Hooks must always be called unconditionally at the top level of the component.

</details>

---

### Q2

```jsx
function App() {
  const items = [1, 2, 3];

  items.forEach((item) => {
    const [value, setValue] = useState(item);
  });

  return <h1>App</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

React throws an error:

```txt
React Hook "useState" cannot be called inside a callback.
React Hooks must be called in the same order in every component render.
```

Hooks cannot be called inside loops, callbacks, or nested functions. The call count would vary if the array length changes.

</details>

---

### Q3

```jsx
function App() {
  function handleClick() {
    const [count, setCount] = useState(0);
    setCount(1);
  }

  return <button onClick={handleClick}>Click</button>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

React throws an error:

```txt
Invalid hook call. Hooks can only be called inside of the body of a function component.
```

Hooks cannot be called inside event handlers. They must be called at the top level of the component function.

</details>

---

## 2. useState Basics

### Q4

```jsx
function App() {
  const [count, setCount] = useState(0);

  console.log("render");

  return <h1>{count}</h1>;
}
```

#### ❓ How many times does "render" log on initial mount in production vs development?

<details>
<summary>✅ Answer</summary>

```txt
Production:  render (once)
Development: render (twice)
```

In development with `React.StrictMode`, React intentionally double-invokes the component body to detect side effects. In production, it runs once.

</details>

---

### Q5

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(1);
    console.log(count);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ After clicking, what is logged?

<details>
<summary>✅ Answer</summary>

```txt
0
```

`setState` is asynchronous. `count` is a snapshot of the value from the current render. The logged value is still `0` even after calling `setCount(1)`. The updated value is only accessible in the next render.

</details>

---

### Q6

```jsx
function App() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(0)}>
      {count}
    </button>
  );
}
```

#### ❓ Does clicking the button cause a re-render?

<details>
<summary>✅ Answer</summary>

No.

React uses `Object.is` to compare old and new state before scheduling a re-render. Since `Object.is(0, 0)` is `true`, React bails out and the component does not re-render.

</details>

---

### Q7

```jsx
function App() {
  const [user, setUser] = useState({ name: "Alice", age: 25 });

  function updateAge() {
    user.age = 30;
    setUser(user);
  }

  return <button onClick={updateAge}>{user.age}</button>;
}
```

#### ❓ Does the UI update to show 30 after clicking?

<details>
<summary>✅ Answer</summary>

No (or unreliably).

The object reference passed to `setUser` is the same reference as the current state. `Object.is(user, user)` returns `true`, so React may bail out of re-rendering. Direct mutation also bypasses React's change tracking.

Correct approach:

```jsx
setUser({ ...user, age: 30 });
```

</details>

---

## 3. useEffect Basics

### Q8

```jsx
function App() {
  useEffect(() => {
    console.log("effect");
  });

  console.log("render");

  return <h1>Hello</h1>;
}
```

#### ❓ What is the order of logs on initial mount?

<details>
<summary>✅ Answer</summary>

```txt
render
effect
```

The component function body runs synchronously during the render phase. `useEffect` runs asynchronously after the browser has painted the screen. The render phase always completes before any effects execute.

</details>

---

### Q9

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("effect", count);
  }, []);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

#### ❓ After 3 button clicks, how many times does the effect log in total?

<details>
<summary>✅ Answer</summary>

```txt
effect 0
```

With an empty dependency array, the effect runs exactly once after the initial mount. It logs `count = 0` because the closure captures the value from the first render. The effect never re-runs regardless of how many times `count` changes.

</details>

---

### Q10

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    return () => {
      console.log("cleanup", count);
    };
  }, [count]);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

#### ❓ After clicking once (count goes 0 → 1), what is logged and when?

<details>
<summary>✅ Answer</summary>

```txt
cleanup 0
```

The cleanup function runs before the next effect executes. It closes over the `count` value from the previous render (0). Only after the cleanup runs does the new effect for `count = 1` execute.

</details>

---

### Q11

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  useEffect(() => {
    console.log("effect");
  }, [a]);

  return (
    <>
      <button onClick={() => setA(a + 1)}>A: {a}</button>
      <button onClick={() => setB(b + 1)}>B: {b}</button>
    </>
  );
}
```

#### ❓ Clicking the "B" button — does the effect log "effect"?

<details>
<summary>✅ Answer</summary>

No.

The effect's dependency array only contains `a`. Updating `b` does trigger a re-render, but React checks the deps and finds `a` unchanged, so the effect does not run.

</details>

---

## 4. useRef Basics

### Q12

```jsx
function App() {
  const ref = useRef(0);

  function handleClick() {
    ref.current += 1;
    console.log(ref.current);
  }

  return <button onClick={handleClick}>Click</button>;
}
```

#### ❓ After 3 clicks, what are the console logs and how many re-renders occur?

<details>
<summary>✅ Answer</summary>

```txt
1
2
3
```

Zero re-renders occur.

`useRef` mutations update `ref.current` immediately and the console logs are correct, but mutating `.current` never schedules a re-render. The button label does not update.

</details>

---

### Q13

```jsx
function App() {
  const [count, setCount] = useState(0);
  const prevCount = useRef(0);

  useEffect(() => {
    prevCount.current = count;
  });

  return (
    <p>
      Now: {count}, Before: {prevCount.current}
    </p>
  );
}
```

#### ❓ After clicking increment once (count = 1), what does the paragraph show?

<details>
<summary>✅ Answer</summary>

```txt
Now: 1, Before: 0
```

During the render where `count = 1`, the effect from the previous render has already run and stored `prevCount.current = 0`. The new effect runs after this render and updates `prevCount.current = 1` — but that update is not visible until the next render.

</details>

---

### Q14

```jsx
function App() {
  const inputRef = useRef(null);

  useEffect(() => {
    console.log(inputRef.current);
  }, []);

  return <input ref={inputRef} />;
}
```

#### ❓ What is logged?

<details>
<summary>✅ Answer</summary>

```txt
<input>  (the actual DOM element)
```

After the initial mount, `inputRef.current` is set to the real DOM `<input>` node. The effect runs after mount, so the ref is already populated.

</details>

---

## 5. useMemo and useCallback

### Q15

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(0);

  const doubled = useMemo(() => {
    console.log("computing");
    return count * 2;
  }, [count]);

  return (
    <>
      <p>{doubled}</p>
      <button onClick={() => setCount(count + 1)}>Count</button>
      <button onClick={() => setOther(other + 1)}>Other</button>
    </>
  );
}
```

#### ❓ Clicking "Other" — does "computing" log?

<details>
<summary>✅ Answer</summary>

No.

Clicking "Other" updates `other` and triggers a re-render, but `useMemo` checks its dependency array `[count]`. Since `count` has not changed, the memoized computation is skipped and the cached value is returned.

</details>

---

### Q16

```jsx
function App() {
  const [count, setCount] = useState(0);

  const objA = useMemo(() => ({ value: 1 }), []);
  const objB = { value: 1 };

  console.log(objA === objB);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

#### ❓ What is logged on every click?

<details>
<summary>✅ Answer</summary>

```txt
false
```

`objA` is the same object reference on every render (memoized with empty deps). `objB` is a new object literal created on every render. Even though their contents are identical, `===` compares by reference, not by value.

</details>

---

### Q17

```jsx
const Child = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>Child</button>;
});

function App() {
  const [count, setCount] = useState(0);

  const handleClick = () => console.log("clicked");

  return (
    <>
      <Child onClick={handleClick} />
      <button onClick={() => setCount(count + 1)}>{count}</button>
    </>
  );
}
```

#### ❓ Does Child re-render when the count button is clicked?

<details>
<summary>✅ Answer</summary>

Yes.

`handleClick` is a new function reference on every render of `App`. `React.memo` uses shallow comparison — the `onClick` prop appears changed, so Child re-renders despite having no functional difference.

Fix:

```jsx
const handleClick = useCallback(() => console.log("clicked"), []);
```

</details>

---

### Q18

```jsx
function App() {
  const [count, setCount] = useState(0);

  const memoizedValue = useMemo(() => count * 2, [count]);

  useEffect(() => {
    console.log("effect", memoizedValue);
  }, [memoizedValue]);

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

#### ❓ Does the effect run every time count changes?

<details>
<summary>✅ Answer</summary>

Yes.

When `count` changes, `useMemo` recomputes `memoizedValue` to a new number. `useEffect` sees that `memoizedValue` changed (by `Object.is` comparison) and runs the effect.

</details>

---

## 6. useContext

### Q19

```jsx
const CountContext = createContext(0);

function Child() {
  const count = useContext(CountContext);
  console.log("Child render");
  return <p>{count}</p>;
}

function App() {
  const [count, setCount] = useState(0);

  return (
    <CountContext.Provider value={count}>
      <Child />
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </CountContext.Provider>
  );
}
```

#### ❓ Does Child re-render on every button click?

<details>
<summary>✅ Answer</summary>

Yes.

Every time `count` changes, the Provider receives a new value. All consumers of `CountContext` — including `Child` — re-render regardless of where they are in the tree.

</details>

---

### Q20

```jsx
const ThemeContext = createContext("light");

function Child() {
  const theme = useContext(ThemeContext);
  return <p>{theme}</p>;
}

function App() {
  return <Child />;
}
```

#### ❓ What does Child render?

<details>
<summary>✅ Answer</summary>

```txt
light
```

No Provider exists in the tree. When no Provider is found, `useContext` falls back to the default value supplied to `createContext`.

</details>

---

### Q21

```jsx
const Ctx = createContext(null);

function Child() {
  const value = useContext(Ctx);
  console.log("Child render", value);
  return <p>{value}</p>;
}

function App() {
  const [val, setVal] = useState(1);

  return (
    <Ctx.Provider value={val}>
      <Child />
      <button onClick={() => setVal(1)}>Set Same</button>
    </Ctx.Provider>
  );
}
```

#### ❓ Does Child re-render when "Set Same" is clicked?

<details>
<summary>✅ Answer</summary>

No.

`setVal(1)` sets state to the same value. React bails out and does not re-render `App`, so the Provider does not receive a new value and context consumers do not re-render.

</details>

---

## 7. Mixed Hooks

### Q22

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log("A");
  }, []);

  useEffect(() => {
    console.log("B", count);
  }, [count]);

  console.log("render");

  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

#### ❓ On initial mount, what is the exact log order?

<details>
<summary>✅ Answer</summary>

```txt
render
A
B 0
```

The render phase (function body) runs first and synchronously. After the browser paints, effects run in the order they are declared in the component.

</details>

---

### Q23

```jsx
function App() {
  const [count, setCount] = useState(0);
  const ref = useRef(count);

  useEffect(() => {
    ref.current = count;
  });

  function handleLog() {
    setTimeout(() => {
      console.log("state:", count);
      console.log("ref:", ref.current);
    }, 2000);
  }

  return (
    <>
      <button onClick={handleLog}>Log</button>
      <button onClick={() => setCount(count + 1)}>Inc</button>
    </>
  );
}
```

#### ❓ If you click Log (count = 0) and then Inc twice within 2 seconds, what is logged after 2s?

<details>
<summary>✅ Answer</summary>

```txt
state: 0
ref: 2
```

The closure in `setTimeout` captures `count = 0` at the time `handleLog` was called. However, `ref.current` is updated by the effect after each render, so it always holds the latest value. After two increments, `ref.current` is `2`.

</details>

---

### Q24

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  const sum = useMemo(() => a + b, [a, b]);

  return (
    <>
      <p>Sum: {sum}</p>
      <button onClick={() => setA(a + 1)}>A: {a}</button>
      <button onClick={() => setB(b + 1)}>B: {b}</button>
    </>
  );
}
```

#### ❓ After clicking "A" once and "B" once (separately), what does sum display?

<details>
<summary>✅ Answer</summary>

```txt
Sum: 2
```

Each click triggers a separate render. After "A": `a = 1, b = 0, sum = 1`. After "B": `a = 1, b = 1, sum = 2`. `useMemo` recomputes on both renders because a dependency changed each time.

</details>

---

### Q25

```jsx
function useCounter(initial) {
  const [count, setCount] = useState(initial);
  const increment = useCallback(() => setCount(c => c + 1), []);
  return { count, increment };
}

function App() {
  const { count, increment } = useCounter(0);
  const { count: count2, increment: increment2 } = useCounter(10);

  return (
    <>
      <button onClick={increment}>{count}</button>
      <button onClick={increment2}>{count2}</button>
    </>
  );
}
```

#### ❓ Are `count` and `count2` shared? Does incrementing one affect the other?

<details>
<summary>✅ Answer</summary>

No — they are completely independent.

Each call to a custom hook creates its own isolated hook instances. `useCounter(0)` and `useCounter(10)` each have their own `useState` slot in `App`'s fiber. Incrementing one has no effect on the other.

</details>

---

## ✅ Topics Covered

- Rules of Hooks (conditional, loop, callback violations)
- useState snapshot behavior
- setState async timing
- Same-value state bail-out
- Direct object mutation detection
- useEffect timing relative to render phase
- useEffect with empty vs populated dependency array
- useEffect cleanup order and closure capture
- useRef no re-render on mutation
- useRef for previous value pattern
- DOM ref after mount
- useMemo memoization and dependency checking
- Referential equality with useMemo
- React.memo combined with useCallback
- Context default value fallback
- Context re-render propagation
- Context same-value optimization
- Multiple effects execution order
- Ref vs closure for latest value access
- Custom hook state isolation
