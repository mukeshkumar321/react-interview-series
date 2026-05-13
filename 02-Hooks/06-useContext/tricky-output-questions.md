## useContext — Tricky Output Questions

> These questions test your understanding of context value propagation, re-render behavior, context with useState and useReducer, and common edge cases. Each question reflects real scenarios from senior React interviews.

---

## 1. Basic Context

### Q1

```jsx
const CountContext = createContext(0);

function Display() {
  const count = useContext(CountContext);
  console.log('Display rendered:', count);
  return <p>{count}</p>;
}

function App() {
  return (
    <CountContext.Provider value={5}>
      <Display />
    </CountContext.Provider>
  );
}
```

#### ❓ What is logged when this renders?

<details>
<summary>✅ Answer</summary>

```txt
Display rendered: 5
```

**Explanation:** `useContext(CountContext)` returns the current value of the nearest `CountContext.Provider` above the consumer in the tree. The `<CountContext.Provider value={5}>` provides `5` to all descendants. `Display` consumes the context and receives `5`. The default value (`0`) is only used when there is no matching Provider above in the tree.

</details>

---

### Q2

```jsx
const ThemeContext = createContext('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  console.log('theme:', theme);
  return <button>{theme}</button>;
}

function App() {
  return <ThemedButton />;
  // No Provider!
}
```

#### ❓ What is logged?

<details>
<summary>✅ Answer</summary>

```txt
theme: light
```

**Explanation:** When a component calls `useContext` but there is no matching Provider above it in the tree, the context returns its **default value** — the argument passed to `createContext`. Here, `createContext('light')` sets the default to `'light'`. The default value is used when there is no Provider, not when the Provider value is `undefined`.

</details>

---

### Q3

```jsx
const ValueContext = createContext('outer');

function Inner() {
  const value = useContext(ValueContext);
  console.log('inner:', value);
  return <p>{value}</p>;
}

function Middle() {
  return (
    <ValueContext.Provider value="middle">
      <Inner />
    </ValueContext.Provider>
  );
}

function App() {
  return (
    <ValueContext.Provider value="outer">
      <Middle />
    </ValueContext.Provider>
  );
}
```

#### ❓ What does `Inner` display?

<details>
<summary>✅ Answer</summary>

```txt
inner: middle
```

**Explanation:** `useContext` returns the value from the **nearest** ancestor Provider in the component tree. `Inner` is a child of `Middle`, which provides `"middle"`. Even though `App` also provides a `ValueContext` with `"outer"`, the closer `Middle` provider wins. Nested Providers shadow outer ones for all components within their subtree.

</details>

---

### Q4

```jsx
const DataContext = createContext(null);

function Consumer() {
  const data = useContext(DataContext);
  console.log('data:', data);
  return <p>{data?.name ?? 'no data'}</p>;
}

function App() {
  return (
    <DataContext.Provider value={undefined}>
      <Consumer />
    </DataContext.Provider>
  );
}
```

#### ❓ What is logged? Does the Provider value override the default value?

<details>
<summary>✅ Answer</summary>

```txt
data: undefined
```

**Explanation:** When a Provider explicitly passes `value={undefined}`, the consumer receives `undefined` — NOT the default value from `createContext(null)`. The default value is only used when there is **no Provider at all** in the ancestor tree. An explicit `value={undefined}` is still a valid Provider value and overrides the default. The `<p>` displays "no data" because `undefined?.name` is `undefined` and the nullish coalescing falls back to `'no data'`.

</details>

---

### Q5

```jsx
const CountContext = createContext(0);

function Counter() {
  const count = useContext(CountContext);
  return <p>Count: {count}</p>;
}

function App() {
  const [value, setValue] = useState(10);

  return (
    <CountContext.Provider value={value}>
      <Counter />
      <button onClick={() => setValue(v => v + 1)}>+</button>
    </CountContext.Provider>
  );
}
```

#### ❓ What is displayed initially? After clicking "+"?

<details>
<summary>✅ Answer</summary>

```txt
// Initial: Count: 10
// After click: Count: 11
```

**Explanation:** The Provider's `value` is tied to `App`'s state via `value={value}`. When `setValue` is called, `App` re-renders, passing the new `value` to the Provider. React detects the context value changed and re-renders all `useContext(CountContext)` consumers — including `Counter`. Context is not static; it is reactive when connected to state.

</details>

---

## 2. Re-render Behavior

### Q6

```jsx
const ThemeContext = createContext('light');

function Header() {
  const theme = useContext(ThemeContext);
  console.log('Header rendered');
  return <header>{theme}</header>;
}

function Sidebar() {
  console.log('Sidebar rendered');
  return <aside>Sidebar</aside>;
}

function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <Sidebar />
      <button onClick={() => setTheme('dark')}>Dark</button>
    </ThemeContext.Provider>
  );
}
```

#### ❓ After clicking "Dark", which components re-render?

<details>
<summary>✅ Answer</summary>

```txt
App re-renders (state change).
Header re-renders (consumes the changed context).
Sidebar re-renders (child of App — re-renders because App re-rendered).
```

**Explanation:** When `App`'s state changes, `App` re-renders. All of `App`'s children re-render by default (unless wrapped in `React.memo`). `Header` re-renders both because it is a child of `App` AND because it consumes the changed context value. `Sidebar` re-renders because it is a plain child of `App` (no memoization). Context change alone would only trigger re-renders for consumers, but here the parent state change causes all children to re-render anyway.

</details>

---

### Q7

```jsx
const CountContext = createContext(0);

const Child = React.memo(() => {
  console.log('Child rendered');
  return <p>I do not consume context</p>;
});

function App() {
  const [count, setCount] = useState(0);

  return (
    <CountContext.Provider value={count}>
      <Child />
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </CountContext.Provider>
  );
}
```

#### ❓ After clicking "+" 3 times, how many times does "Child rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only the initial render)
```

**Explanation:** `Child` is wrapped in `React.memo` and receives no props. Even though `App` re-renders on every click, `React.memo` protects `Child` from re-rendering when its props haven't changed. Crucially, `Child` does NOT call `useContext(CountContext)` — it does not subscribe to the context. A component only re-renders due to context changes if it explicitly calls `useContext` for that context. Wrapping in `React.memo` + not consuming context = zero re-renders from parent state changes.

</details>

---

### Q8

```jsx
const UserContext = createContext({ name: 'Alice' });

function UserDisplay() {
  const user = useContext(UserContext);
  console.log('UserDisplay rendered');
  return <p>{user.name}</p>;
}

function App() {
  const [count, setCount] = useState(0);

  return (
    <UserContext.Provider value={{ name: 'Alice' }}>
      <UserDisplay />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </UserContext.Provider>
  );
}
```

#### ❓ After clicking the button 3 times, how many times does "UserDisplay rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
4 times (initial render + 3 button clicks)
```

**Explanation:** Every time `App` re-renders (due to `count` state change), a new `{ name: 'Alice' }` object is created inline. React compares the old context value with the new one using `Object.is`. Two different objects with the same shape are NOT equal by reference. So React sees the context value as "changed" on every render, causing `UserDisplay` to re-render every time. Fix: memoize the context value with `useMemo(() => ({ name: 'Alice' }), [])` or store it in state.

</details>

---

### Q9

```jsx
const CountContext = createContext(0);
const SetCountContext = createContext(() => {});

function Counter() {
  const count = useContext(CountContext);
  console.log('Counter rendered');
  return <p>{count}</p>;
}

function Controls() {
  const setCount = useContext(SetCountContext);
  console.log('Controls rendered');
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}

function App() {
  const [count, setCount] = useState(0);

  return (
    <CountContext.Provider value={count}>
      <SetCountContext.Provider value={setCount}>
        <Counter />
        <Controls />
      </SetCountContext.Provider>
    </CountContext.Provider>
  );
}
```

#### ❓ After clicking "+", which components re-render?

<details>
<summary>✅ Answer</summary>

```txt
App re-renders (state change).
Counter re-renders (CountContext value changed: 0 → 1).
Controls does NOT re-render due to context change (SetCountContext value is the same function reference).
But Controls may re-render because App re-rendered and Controls is a child.
```

**Explanation:** React's `useState` returns a stable `setCount` function (same reference across renders). `SetCountContext.Provider value={setCount}` provides the same function reference on every render — `Object.is` returns `true` → no context-triggered re-render for `Controls`. However, `Controls` still re-renders because it is a direct child of `App` (no `React.memo`). Splitting contexts is still useful: with `React.memo` on `Controls`, it would bail out entirely. Without it, parent re-renders cascade regardless.

</details>

---

### Q10

```jsx
const ThemeContext = createContext('light');

const ThemeDisplay = React.memo(() => {
  const theme = useContext(ThemeContext);
  console.log('ThemeDisplay rendered');
  return <p>{theme}</p>;
});

function App() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <ThemeDisplay />
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={() => setTheme('dark')}>Dark</button>
    </ThemeContext.Provider>
  );
}
```

#### ❓ "Count" is clicked 3 times, then "Dark" is clicked once. How many times does "ThemeDisplay rendered" log total?

<details>
<summary>✅ Answer</summary>

```txt
2 times: initial render + 1 after "Dark"
```

**Explanation:** `ThemeDisplay` is wrapped in `React.memo` and receives no props. When "Count" is clicked, `App` re-renders. `React.memo` checks props — none changed. BUT `ThemeDisplay` calls `useContext(ThemeContext)`. When `theme` is unchanged, the context value is the same string (`'light'`). `Object.is('light', 'light') = true` — no context re-render. `React.memo` bails out from the parent re-render too. When "Dark" is clicked, context value changes → `useContext` triggers a re-render despite `React.memo`.

**Key insight:** `React.memo` does NOT prevent re-renders caused by context changes. Context change bypasses `React.memo`.

</details>

---

## 3. Context + useState

### Q11

```jsx
const CountContext = createContext();

function CountProvider({ children }) {
  const [count, setCount] = useState(0);
  return (
    <CountContext.Provider value={{ count, setCount }}>
      {children}
    </CountContext.Provider>
  );
}

function Counter() {
  const { count } = useContext(CountContext);
  console.log('Counter rendered:', count);
  return <p>{count}</p>;
}

function Controls() {
  const { setCount } = useContext(CountContext);
  console.log('Controls rendered');
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}

function App() {
  return (
    <CountProvider>
      <Counter />
      <Controls />
    </CountProvider>
  );
}
```

#### ❓ After clicking "+" 3 times, how many times does "Controls rendered" log total?

<details>
<summary>✅ Answer</summary>

```txt
4 times (initial + 3 clicks)
```

**Explanation:** The context value is `{ count, setCount }` — a new object created on every render of `CountProvider`. Every time `count` changes, `CountProvider` re-renders, creating a new `{ count, setCount }` object. `Object.is(oldValue, newValue) = false` → ALL context consumers re-render, including `Controls` (even though `setCount` itself is stable). To prevent `Controls` from re-rendering unnecessarily, split the context into a value context and a dispatch context.

</details>

---

### Q12

```jsx
const CountContext = createContext();

function CountProvider({ children }) {
  const [count, setCount] = useState(0);
  const value = useMemo(() => ({ count, setCount }), [count]);
  return (
    <CountContext.Provider value={value}>
      {children}
    </CountContext.Provider>
  );
}

function Controls() {
  const { setCount } = useContext(CountContext);
  console.log('Controls rendered');
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}

function App() {
  return (
    <CountProvider>
      <Controls />
    </CountProvider>
  );
}
```

#### ❓ After clicking "+" 3 times, how many times does "Controls rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
4 times (initial + 3 clicks) — same as before!
```

**Explanation:** `useMemo(() => ({ count, setCount }), [count])` creates a new object every time `count` changes. When `count` changes (every click), the memoized value is recomputed → new object reference → context value changes → all consumers re-render including `Controls`. `useMemo` here only prevents re-renders when `count` does NOT change. When count DOES change, `Controls` still re-renders. The real fix is to split into two contexts: one for the value and one for the setter.

</details>

---

### Q13

```jsx
const ValueContext = createContext(null);
const SetterContext = createContext(null);

function Provider({ children }) {
  const [count, setCount] = useState(0);
  return (
    <ValueContext.Provider value={count}>
      <SetterContext.Provider value={setCount}>
        {children}
      </SetterContext.Provider>
    </ValueContext.Provider>
  );
}

const Controls = React.memo(() => {
  const setCount = useContext(SetterContext);
  console.log('Controls rendered');
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
});

function Counter() {
  const count = useContext(ValueContext);
  console.log('Counter rendered:', count);
  return <p>{count}</p>;
}

function App() {
  return (
    <Provider>
      <Counter />
      <Controls />
    </Provider>
  );
}
```

#### ❓ After clicking "+" 3 times, how many times do "Counter rendered" and "Controls rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
Counter rendered: 4 times (initial + 3 clicks)
Controls rendered: 1 time (initial render only)
```

**Explanation:** Split contexts are the key. `ValueContext` provides `count` (changes on each click). `SetterContext` provides `setCount` (stable reference — same function from `useState`). `Counter` subscribes to `ValueContext` — re-renders every time count changes. `Controls` (wrapped in `React.memo`) subscribes to `SetterContext` — the setter is always the same reference, so context doesn't trigger a re-render. `React.memo` prevents the parent re-render from cascading down. This is the split context pattern for performance.

</details>

---

### Q14

```jsx
const CountContext = createContext(0);

function useCount() {
  return useContext(CountContext);
}

function Display() {
  const count = useCount();
  return <p>{count}</p>;
}

function App() {
  const [count, setCount] = useState(5);

  return (
    <CountContext.Provider value={count}>
      <Display />
      <button onClick={() => setCount(10)}>Set to 10</button>
      <button onClick={() => setCount(10)}>Set to 10 again</button>
    </CountContext.Provider>
  );
}
```

#### ❓ After clicking "Set to 10", then "Set to 10 again" — does `Display` re-render the second time?

<details>
<summary>✅ Answer</summary>

```txt
No — Display does NOT re-render when "Set to 10 again" is clicked.
```

**Explanation:** After the first "Set to 10" click, `count` becomes `10`. The second click calls `setCount(10)` when `count` is already `10`. `Object.is(10, 10) = true` → React bails out — no re-render of `App`. Since `App` doesn't re-render, the Provider doesn't get a new value, and `Display` doesn't re-render. Context consumers only re-render when the context value actually changes.

</details>

---

### Q15

```jsx
const AlertContext = createContext(null);

function Alert() {
  const { message } = useContext(AlertContext);
  if (!message) return null;
  return <div className="alert">{message}</div>;
}

function TriggerButton() {
  const { setMessage } = useContext(AlertContext);
  return (
    <button onClick={() => setMessage('Error occurred!')}>Trigger Alert</button>
  );
}

function App() {
  const [message, setMessage] = useState(null);

  return (
    <AlertContext.Provider value={{ message, setMessage }}>
      <Alert />
      <TriggerButton />
    </AlertContext.Provider>
  );
}
```

#### ❓ After clicking "Trigger Alert", what happens to `Alert` and `TriggerButton`?

<details>
<summary>✅ Answer</summary>

```txt
Both Alert and TriggerButton re-render.
Alert now renders the <div class="alert">Error occurred!</div> element.
TriggerButton re-renders (context value object changed), though its output is unchanged.
```

**Explanation:** `setMessage('Error occurred!')` updates `App`'s state. `App` re-renders, creating a new `{ message, setMessage }` object. All context consumers receive a new context value (new object reference) and re-render. `Alert` now has `message = 'Error occurred!'` and renders the alert div. `TriggerButton` re-renders unnecessarily because the context value is a new object even though `setMessage` is stable. This is the common pattern where context performance isn't optimized.

</details>

---

## 4. Context + useReducer

### Q16

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 };
    case 'RESET': return { count: 0 };
    default: return state;
  }
};

const StoreContext = createContext(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <StoreContext.Provider value={{ state, dispatch }}>
      {children}
    </StoreContext.Provider>
  );
}

function Counter() {
  const { state } = useContext(StoreContext);
  console.log('Counter:', state.count);
  return <p>{state.count}</p>;
}

function Controls() {
  const { dispatch } = useContext(StoreContext);
  console.log('Controls rendered');
  return (
    <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
  );
}
```

#### ❓ After clicking "+" once, how many times do "Counter" and "Controls" log?

<details>
<summary>✅ Answer</summary>

```txt
Counter: 2 times (initial + click)
Controls: 2 times (initial + click)
```

**Explanation:** Dispatching `INCREMENT` changes the `state` object. `Provider` re-renders with new state. The context value `{ state, dispatch }` is a new object (even though `dispatch` is stable — same reference from `useReducer`). Because the context value is a new object, ALL consumers re-render: both `Counter` (value changed) and `Controls` (object reference changed even though dispatch is stable). For optimization, split into two contexts: one for state, one for dispatch.

</details>

---

### Q17

```jsx
const StateContext = createContext(null);
const DispatchContext = createContext(null);

const reducer = (state, action) => {
  if (action.type === 'INC') return { count: state.count + 1 };
  return state;
};

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  );
}

const Controls = React.memo(() => {
  const dispatch = useContext(DispatchContext);
  console.log('Controls rendered');
  return <button onClick={() => dispatch({ type: 'INC' })}>+</button>;
});

function Counter() {
  const state = useContext(StateContext);
  return <p>{state.count}</p>;
}
```

#### ❓ After clicking "+" 5 times, how many times does "Controls rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only the initial render)
```

**Explanation:** `dispatch` from `useReducer` is stable — same function reference across all renders (React guarantees this). `DispatchContext.Provider value={dispatch}` provides the same reference every render. `Controls` (wrapped in `React.memo`) consumes `DispatchContext`. `Object.is(dispatch, dispatch) = true` → no context re-render. `React.memo` also prevents parent re-render from cascading. This split-context + `React.memo` + `useReducer` pattern is the gold standard for context performance.

</details>

---

### Q18

```jsx
const DispatchContext = createContext(null);

const reducer = (state, action) => {
  switch (action.type) {
    case 'LOG': console.log('reducing LOG, payload:', action.payload); return state;
    default: return state;
  }
};

function App() {
  const [state, dispatch] = useReducer(reducer, {});

  return (
    <DispatchContext.Provider value={dispatch}>
      <Child />
    </DispatchContext.Provider>
  );
}

function Child() {
  const dispatch = useContext(DispatchContext);

  useEffect(() => {
    dispatch({ type: 'LOG', payload: 'mounted' });
  }, [dispatch]);

  return <p>Child</p>;
}
```

#### ❓ What is logged when the component mounts?

<details>
<summary>✅ Answer</summary>

```txt
reducing LOG, payload: mounted
```

**Explanation:** On mount, `Child`'s `useEffect` fires and calls `dispatch({ type: 'LOG', payload: 'mounted' })`. The reducer logs `'mounted'` and returns the same state (no state change). The `dispatch` function is stable from `useReducer`, so the `[dispatch]` dep in `useEffect` never changes and the effect runs only once. This is a valid pattern for dispatching a side-effect action on mount.

</details>

---

### Q19

```jsx
const CountContext = createContext(0);
const DispatchContext = createContext(null);

const reducer = (count, action) => {
  if (action.type === 'SET') return action.value;
  return count;
};

function Provider({ children }) {
  const [count, dispatch] = useReducer(reducer, 0);
  return (
    <CountContext.Provider value={count}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </CountContext.Provider>
  );
}

function Display() {
  const count = useContext(CountContext);
  console.log('Display:', count);
  return <p>{count}</p>;
}

function Setter() {
  const dispatch = useContext(DispatchContext);
  return (
    <button onClick={() => dispatch({ type: 'SET', value: 10 })}>
      Set to 10
    </button>
  );
}
```

#### ❓ After clicking "Set to 10" twice, how many times does "Display" log total?

<details>
<summary>✅ Answer</summary>

```txt
2 times: initial render (0) + after first click (10).
Second click does NOT cause a re-render.
```

**Explanation:** After the first click, `count = 10`. The second click dispatches `{ type: 'SET', value: 10 }`. The reducer returns `10` — the same value as the current state. `Object.is(10, 10) = true` → React bails out. No re-render of `Provider`, no new context value, no re-render of `Display`. React's bailout optimization applies to `useReducer` too: if the new state is the same (by `Object.is`), no re-render occurs.

</details>

---

### Q20

```jsx
const ThemeContext = createContext('light');

function App() {
  const theme = useContext(ThemeContext);
  console.log('App theme:', theme);
  return <p>{theme}</p>;
}

// Rendered as:
ReactDOM.createRoot(document.getElementById('root')).render(
  <ThemeContext.Provider value="dark">
    <App />
  </ThemeContext.Provider>
);
```

#### ❓ What is logged?

<details>
<summary>✅ Answer</summary>

```txt
App theme: dark
```

**Explanation:** `App` calls `useContext(ThemeContext)` and is rendered inside a `<ThemeContext.Provider value="dark">`. The nearest Provider provides `'dark'`, overriding the default `'light'`. The Provider can be placed anywhere in the component tree — including at the root level in the `render` call. `App` correctly receives `'dark'` from its Provider ancestor.

</details>

---

## 5. Edge Cases

### Q21

```jsx
const CountContext = createContext(0);

function Counter() {
  const count = useContext(CountContext);
  console.log('Counter rendered');
  return <p>{count}</p>;
}

function App() {
  const [show, setShow] = useState(true);

  return (
    <>
      <CountContext.Provider value={42}>
        {show && <Counter />}
      </CountContext.Provider>
      <button onClick={() => setShow(false)}>Hide</button>
    </>
  );
}
```

#### ❓ After clicking "Hide", does `Counter` try to read the context one last time before unmounting?

<details>
<summary>✅ Answer</summary>

```txt
No — Counter does not re-render or read context during unmount.
```

**Explanation:** When `show` becomes `false`, `Counter` is removed from the tree entirely. React unmounts it without re-rendering it. `useContext` does not fire during unmount. If `Counter` had a `useEffect` with cleanup, the cleanup would run — but that's unrelated to context reading. The component simply ceases to exist in the tree when the condition is false.

</details>

---

### Q22

```jsx
const Ctx = createContext('default');

const Memoized = React.memo(({ label }) => {
  const value = useContext(Ctx);
  console.log('Memoized rendered, value:', value);
  return <p>{label}: {value}</p>;
});

function App() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState('light');

  return (
    <Ctx.Provider value={theme}>
      <Memoized label="theme-display" />
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={() => setTheme('dark')}>Dark</button>
    </Ctx.Provider>
  );
}
```

#### ❓ "Count" is clicked 3 times, then "Dark" is clicked once. How many times does "Memoized rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
2 times: initial render + after "Dark" click.
"Count" clicks do NOT cause "Memoized" to re-render.
```

**Explanation:** `React.memo` protects `Memoized` from parent re-renders when props don't change. `label` is always `"theme-display"` — stable. When "Count" is clicked, `App` re-renders but `Memoized`'s props are unchanged AND the context value (`theme = 'light'`) hasn't changed → `React.memo` bails out. When "Dark" is clicked, the context value changes to `'dark'`. Context changes bypass `React.memo` — the component re-renders to reflect the new context value.

</details>

---

### Q23

```jsx
const Ctx = createContext(null);

function useCtxValue() {
  const ctx = useContext(Ctx);
  if (ctx === null) {
    throw new Error('useCtxValue must be used within a Provider');
  }
  return ctx;
}

function Child() {
  const value = useCtxValue();
  return <p>{value}</p>;
}

function App() {
  return <Child />;  // No Provider
}
```

#### ❓ What happens when this renders?

<details>
<summary>✅ Answer</summary>

```txt
React throws an error: "useCtxValue must be used within a Provider"
```

**Explanation:** `createContext(null)` sets the default to `null`. When `Child` renders without a Provider, `useContext(Ctx)` returns `null`. The custom hook `useCtxValue` checks for `null` and throws. This is the "guard hook" pattern — a custom hook that wraps `useContext` and throws a helpful error message if used outside its Provider. This is better than receiving `null` silently and getting a confusing `Cannot read property of null` error later.

</details>

---

### Q24

```jsx
const CountContext = createContext(0);

function DeepChild() {
  const count = useContext(CountContext);
  console.log('DeepChild:', count);
  return <p>{count}</p>;
}

function Middle({ children }) {
  console.log('Middle rendered');
  return <div>{children}</div>;
}

function App() {
  const [count, setCount] = useState(0);

  return (
    <CountContext.Provider value={count}>
      <Middle>
        <DeepChild />
      </Middle>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </CountContext.Provider>
  );
}
```

#### ❓ After clicking "+", does `Middle` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — Middle re-renders.
```

**Explanation:** `Middle` is a direct child of `App`. When `App`'s state changes, React re-renders `App` and by default re-renders all its children — including `Middle`. Even though `Middle` does not consume the context, it re-renders because its parent re-rendered (no `React.memo`). `DeepChild` re-renders both because it's a descendant being re-rendered AND because it consumes the changed context. To prevent `Middle` from re-rendering, wrap it in `React.memo` or lift children composition.

</details>

---

### Q25

```jsx
const Ctx = createContext('v1');

function App() {
  const [version, setVersion] = useState('v1');

  return (
    <Ctx.Provider value={version}>
      <Ctx.Provider value="override">
        <Consumer />
      </Ctx.Provider>
    </Ctx.Provider>
  );
}

function Consumer() {
  const val = useContext(Ctx);
  console.log('val:', val);
  return <p>{val}</p>;
}
```

#### ❓ What does `Consumer` display? Does clicking a button that changes `version` to `"v2"` affect `Consumer`?

<details>
<summary>✅ Answer</summary>

```txt
val: override
Changing version to "v2" does NOT affect Consumer.
```

**Explanation:** `Consumer` is inside the inner `<Ctx.Provider value="override">`. `useContext` always uses the nearest ancestor Provider. The inner Provider hardcodes `"override"` — it does not read from the outer Provider's value at all. Even when the outer Provider's value changes (via `version` state), the inner Provider shields `Consumer` from that change. `Consumer` always sees `"override"`.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Basic Context | Q1 – Q5 | useContext return value, default value usage, nearest Provider wins, undefined vs no Provider, reactive context |
| Re-render Behavior | Q6 – Q10 | which components re-render on context change, React.memo + non-consumer, object reference instability, stable setter reference, React.memo + context bypass |
| Context + useState | Q11 – Q15 | combined value object causes all consumers to re-render, useMemo on context value, split contexts + React.memo, same-value bailout, new object on setState |
| Context + useReducer | Q16 – Q20 | dispatch from context, stable dispatch reference, split contexts optimization, reducer bailout on same state, root-level Provider |
| Edge Cases | Q21 – Q25 | unmount does not read context, React.memo + context change bypass, guard hook pattern, Middle component re-renders, nested same-context providers |
