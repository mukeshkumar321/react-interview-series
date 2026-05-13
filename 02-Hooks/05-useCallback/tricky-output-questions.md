## 📚 useCallback — Tricky Output Questions

> Focus: referential equality, React.memo interaction, stale closures, dependency array pitfalls, useCallback vs useMemo equivalence, and edge cases with useReducer and custom hooks.

---

## 1. Reference Equality

### Q1

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const fn1 = () => {};
  const fn2 = React.useCallback(() => {}, []);

  console.log(fn1 === fn1); // (a)
  console.log(fn2 === fn2); // (b)

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
(a) true
(b) true
```

Both comparisons are within the same render. `fn1` is compared to itself — same reference in the same execution. `fn2` is also compared to itself. The difference only shows across renders. Within one render, every variable reference is consistent.

</details>

---

### Q2

```jsx
let prevFn1, prevFn2;

function App() {
  const [count, setCount] = React.useState(0);

  const fn1 = () => {};
  const fn2 = React.useCallback(() => {}, []);

  if (prevFn1 !== undefined) {
    console.log('fn1 same?', fn1 === prevFn1); // (a)
    console.log('fn2 same?', fn2 === prevFn2); // (b)
  }

  prevFn1 = fn1;
  prevFn2 = fn2;

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ Output after clicking the button once?

<details>
<summary>✅ Answer</summary>

```txt
fn1 same? false
fn2 same? true
```

`fn1` is recreated on every render — a new function object is allocated each time the component function runs. `fn2` is memoized with an empty dependency array, so `useCallback` returns the same function reference across renders as long as no dependencies change.

</details>

---

### Q3

```jsx
function App() {
  const a = React.useCallback(() => {}, []);
  const b = React.useCallback(() => {}, []);

  console.log(a === b);
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
false
```

`a` and `b` are two separate `useCallback` calls. Each one creates and memoizes its own function instance. Even though their source code is identical, they are different objects in memory. `useCallback` only guarantees the same reference across renders for the same hook call — it does not deduplicate functions.

</details>

---

### Q4

```jsx
function App() {
  const [x, setX] = React.useState(0);

  const fn = React.useCallback(() => {
    console.log('x is:', x);
  }, [x]);

  let prevFn = React.useRef(fn);

  React.useEffect(() => {
    console.log('same ref?', prevFn.current === fn);
    prevFn.current = fn;
  });

  return <button onClick={() => setX(v => v + 1)}>inc</button>;
}
```

#### ❓ What is logged on the first button click?

<details>
<summary>✅ Answer</summary>

```txt
same ref? false
```

`x` is in the dependency array. When the button is clicked, `x` changes from `0` to `1`, which triggers a re-render. Because `x` is listed as a dependency, `useCallback` detects that the dependency changed and returns a new function reference. So `prevFn.current` (the old reference) is not equal to the new `fn`.

</details>

---

### Q5

```jsx
function App() {
  const [a, setA] = React.useState(0);
  const [b, setB] = React.useState(0);

  const fn = React.useCallback(() => {
    console.log(a + b);
  }, [a]);

  return (
    <>
      <button onClick={() => setA(v => v + 1)}>inc A</button>
      <button onClick={() => setB(v => v + 1)}>inc B</button>
    </>
  );
}
```

#### ❓ Is a new function reference created when only `b` is incremented?

<details>
<summary>✅ Answer</summary>

```txt
No — same reference returned
```

The dependency array is `[a]`. When `b` increments, `a` is unchanged. `useCallback` compares dependencies using `Object.is` and since `a` did not change, it returns the same memoized function. Note that the callback has a **stale closure** over `b` in this case — it will log the old value of `b`.

</details>

---

## 2. React.memo + useCallback

### Q6

```jsx
let childRenderCount = 0;

const Child = React.memo(({ onClick }) => {
  childRenderCount++;
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = React.useState(0);

  const handleClick = React.useCallback(() => {
    console.log('clicked');
  }, []);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Rerender Parent</button>
      <Child onClick={handleClick} />
    </div>
  );
}
```

#### ❓ What is `childRenderCount` after clicking "Rerender Parent" 3 times?

<details>
<summary>✅ Answer</summary>

```txt
childRenderCount = 1
```

`handleClick` has an empty dependency array, so `useCallback` returns the same reference on every render. `React.memo` compares `onClick` prop: it is the same reference every time, so `Child` re-renders only once — on the initial mount. The 3 parent re-renders do not cause Child to re-render.

</details>

---

### Q7

```jsx
let childRenderCount = 0;

const Child = React.memo(({ onClick }) => {
  childRenderCount++;
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    console.log('clicked');
  };

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Rerender Parent</button>
      <Child onClick={handleClick} />
    </div>
  );
}
```

#### ❓ What is `childRenderCount` after clicking "Rerender Parent" 3 times?

<details>
<summary>✅ Answer</summary>

```txt
childRenderCount = 4
```

`handleClick` is a plain function — a new object is created on every render. `React.memo` compares the `onClick` prop using `===`. Because the reference is always new, the comparison always fails, and `Child` re-renders on every parent render. 1 initial render + 3 triggered = 4 total renders.

</details>

---

### Q8

```jsx
let childRenderCount = 0;

// Note: NOT wrapped in React.memo
const Child = ({ onClick }) => {
  childRenderCount++;
  return <button onClick={onClick}>Click</button>;
};

function Parent() {
  const [count, setCount] = React.useState(0);

  const handleClick = React.useCallback(() => {
    console.log('clicked');
  }, []);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Rerender Parent</button>
      <Child onClick={handleClick} />
    </div>
  );
}
```

#### ❓ What is `childRenderCount` after clicking "Rerender Parent" 3 times?

<details>
<summary>✅ Answer</summary>

```txt
childRenderCount = 4
```

`useCallback` stabilizes `handleClick`, but `Child` is not wrapped in `React.memo`. Without `React.memo`, React never checks props equality — every time the parent renders, the child renders unconditionally. `useCallback` alone cannot prevent child re-renders. Both `React.memo` and `useCallback` are needed.

</details>

---

### Q9

```jsx
let childRenderCount = 0;

const Child = React.memo(({ onClick }) => {
  childRenderCount++;
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = React.useState(0);

  const handleClick = React.useCallback(() => {
    console.log('count:', count);
  }, [count]); // count in deps

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <Child onClick={handleClick} />
    </div>
  );
}
```

#### ❓ What is `childRenderCount` after clicking "Increment" 3 times?

<details>
<summary>✅ Answer</summary>

```txt
childRenderCount = 4
```

`count` is in the dependency array. Every time "Increment" is clicked, `count` changes (0 → 1 → 2 → 3). Each change causes `useCallback` to return a new function reference. `React.memo` sees a different `onClick` prop each time and re-renders `Child`. Initial render (1) + 3 increments (3) = 4 renders total.

</details>

---

### Q10

```jsx
let childRenderCount = 0;

const ExpensiveList = React.memo(({ items, onDelete }) => {
  childRenderCount++;
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          {item.name}
          <button onClick={() => onDelete(item.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
});

function App() {
  const [items, setItems] = React.useState([
    { id: 1, name: 'A' },
    { id: 2, name: 'B' },
  ]);
  const [filter, setFilter] = React.useState('');

  const handleDelete = React.useCallback((id) => {
    setItems(prev => prev.filter(item => item.id !== id));
  }, []);

  return (
    <div>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <ExpensiveList items={items} onDelete={handleDelete} />
    </div>
  );
}
```

#### ❓ Does `ExpensiveList` re-render when the user types in the input?

<details>
<summary>✅ Answer</summary>

```txt
No — ExpensiveList does NOT re-render when filter changes.
```

`handleDelete` has no dependencies (empty array) and never changes reference. The `items` prop also does not change when `filter` changes — only `filter` state changes. `React.memo` compares both `items` (same array reference) and `onDelete` (same function reference). Both comparisons pass, so `ExpensiveList` is skipped. It only re-renders when `items` actually changes (after a delete).

</details>

---

## 3. Stale Callbacks

### Q11

```jsx
function Counter() {
  const [count, setCount] = React.useState(0);

  const getDouble = React.useCallback(() => {
    return count * 2;
  }, []); // empty deps

  React.useEffect(() => {
    const id = setInterval(() => {
      console.log('double:', getDouble());
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty deps

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ What does the interval log after clicking the button 3 times?

<details>
<summary>✅ Answer</summary>

```txt
double: 0
double: 0
double: 0
... (always 0, forever)
```

Both `useCallback` and `useEffect` have empty dependency arrays. `getDouble` captures `count = 0` at mount and never updates. The interval calls `getDouble()` which always returns `0 * 2 = 0`. This is a stale closure. Fix: add `count` to `getDouble`'s deps and `getDouble` to `useEffect`'s deps, or use a ref to hold the latest count.

</details>

---

### Q12

```jsx
function Counter() {
  const [count, setCount] = React.useState(0);

  const increment = React.useCallback(() => {
    setCount(count + 1);
  }, []); // empty deps — stale closure

  return (
    <>
      <p>{count}</p>
      <button onClick={increment}>Increment</button>
    </>
  );
}
```

#### ❓ What happens when the button is clicked repeatedly?

<details>
<summary>✅ Answer</summary>

```txt
count always goes to 1 and stays at 1, regardless of how many times you click.
```

`increment` captures `count = 0` at mount. Every call to `increment` runs `setCount(0 + 1)` — always setting count to `1`. Even after count updates to `1`, the next click still runs `setCount(0 + 1)` because `increment` has the stale `count = 0`. The fix is to use a functional update: `setCount(prev => prev + 1)`.

</details>

---

### Q13

```jsx
function Counter() {
  const [count, setCount] = React.useState(0);

  // Fix applied
  const increment = React.useCallback(() => {
    setCount(prev => prev + 1);
  }, []);

  return (
    <>
      <p>{count}</p>
      <button onClick={increment}>Increment</button>
    </>
  );
}
```

#### ❓ Does this correctly increment count on every click?

<details>
<summary>✅ Answer</summary>

```txt
Yes — count correctly increments by 1 on every click: 0 → 1 → 2 → 3 ...
```

The functional update form `setCount(prev => prev + 1)` does not capture `count` from the closure. Instead, React calls the updater function with the latest state value at the time of the update. This makes `increment` safe to keep with an empty dependency array — no stale closure problem.

</details>

---

### Q14

```jsx
function Messenger({ userId }) {
  const [draft, setDraft] = React.useState('');

  const sendMessage = React.useCallback(() => {
    postMessage(userId, draft);
  }, [userId]); // draft is missing from deps

  return (
    <div>
      <textarea value={draft} onChange={e => setDraft(e.target.value)} />
      <MemoizedSendButton onSend={sendMessage} />
    </div>
  );
}
```

#### ❓ What bug exists here, and when does it manifest?

<details>
<summary>✅ Answer</summary>

```txt
Bug: sendMessage has a stale closure over `draft`.

Manifestation:
  1. User types a message → draft = "Hello World"
  2. sendMessage was created when draft = "" (initial)
  3. useCallback deps: [userId] — draft is not listed
  4. draft changing does NOT cause sendMessage to recreate
  5. Clicking "Send" calls postMessage(userId, "") — sends empty string

Fix: add draft to deps: useCallback(() => postMessage(userId, draft), [userId, draft])
```

</details>

---

## 4. useCallback vs useMemo

### Q15

```jsx
function App() {
  const [n, setN] = React.useState(2);

  const double = React.useCallback(() => n * 2, [n]);
  const tripled = React.useMemo(() => n * 3, [n]);

  console.log(typeof double);   // (a)
  console.log(typeof tripled);  // (b)
  console.log(double());        // (c)
  console.log(tripled);         // (d)
}
```

#### ❓ What are (a), (b), (c), (d)?

<details>
<summary>✅ Answer</summary>

```txt
(a) function
(b) number
(c) 4
(d) 6
```

`useCallback` returns the function itself — `double` is a function, not a number. You must call `double()` to get the computed value. `useMemo` returns the result of calling its factory function — `tripled` is already the computed value `6`, not a function. This is the core distinction between the two hooks.

</details>

---

### Q16

```jsx
function App() {
  const [x, setX] = React.useState(5);

  const fnA = React.useCallback(() => x * 10, [x]);
  const fnB = React.useMemo(() => () => x * 10, [x]);

  console.log(fnA === fnB); // (a)
  console.log(fnA());       // (b)
  console.log(fnB());       // (c)
}
```

#### ❓ What are (a), (b), (c)?

<details>
<summary>✅ Answer</summary>

```txt
(a) false
(b) 50
(c) 50
```

`fnA` and `fnB` are functionally equivalent — they return the same value when called. However, they are different object instances (created by different hook calls), so `fnA === fnB` is `false`. This demonstrates the mathematical equivalence `useCallback(fn, deps) === useMemo(() => fn, deps)` in terms of behavior, but they are not the same object in memory.

</details>

---

### Q17

```jsx
function Component({ items }) {
  const processedItems = React.useCallback(
    () => items.filter(i => i.active).map(i => i.name),
    [items]
  );

  return (
    <ul>
      {processedItems().map(name => <li key={name}>{name}</li>)}
    </ul>
  );
}
```

#### ❓ Does this component have a bug? How should it be written?

<details>
<summary>✅ Answer</summary>

```txt
Yes — wrong hook for the job.

Current behavior:
  processedItems is a FUNCTION, not an array.
  The code calls processedItems() on every render — no memoization benefit.
  The filter+map runs on every render regardless of useCallback.

Fix: use useMemo instead, which returns the computed array directly.

const processedItems = React.useMemo(
  () => items.filter(i => i.active).map(i => i.name),
  [items]
);
// processedItems is now an array, computed only when items changes
```

</details>

---

## 5. Dependency Array

### Q18

```jsx
function SearchBox({ onSearch }) {
  const [query, setQuery] = React.useState('');

  const handleSearch = React.useCallback(() => {
    onSearch(query);
  }, []); // both query and onSearch are missing

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <button onClick={handleSearch}>Search</button>
    </div>
  );
}
```

#### ❓ What are the two bugs, and what is the effect of each?

<details>
<summary>✅ Answer</summary>

```txt
Bug 1: `query` is missing from deps
  Effect: handleSearch always sends the initial query value "" to onSearch,
  regardless of what the user has typed.

Bug 2: `onSearch` is missing from deps
  Effect: if the parent passes a new onSearch function (e.g., the prop changes),
  handleSearch still calls the old version — stale prop reference.

Fix:
  const handleSearch = useCallback(() => {
    onSearch(query);
  }, [query, onSearch]);
```

</details>

---

### Q19

```jsx
function App({ config }) {
  // config is passed as: <App config={{ timeout: 3000 }} />

  const fetchData = React.useCallback(() => {
    requestWithConfig(config);
  }, [config]);

  React.useEffect(() => {
    fetchData();
  }, [fetchData]);
}
```

#### ❓ What problem occurs with this code?

<details>
<summary>✅ Answer</summary>

```txt
Infinite re-render loop (or at minimum, fetchData recreates on every render).

Cause:
  The parent renders <App config={{ timeout: 3000 }} />.
  { timeout: 3000 } is a new object created inline on every parent render.
  Each render: config is a new reference → useCallback deps changed → new fetchData.
  New fetchData → useEffect dep changed → useEffect re-runs → (if it causes state update) → loop.

Fix options:
  1. Memoize config in the parent: const config = useMemo(() => ({ timeout: 3000 }), []);
  2. Pass primitive props instead: <App timeout={3000} />
  3. Use specific fields as deps: useCallback(() => requestWithConfig({ timeout }), [timeout]);
```

</details>

---

### Q20

```jsx
function Logger() {
  const [count, setCount] = React.useState(0);

  const log = React.useCallback(() => {
    console.log('count:', count);
  }, []); // empty deps

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={log}>Log</button>
    </div>
  );
}
```

#### ❓ User clicks Increment 5 times, then clicks Log. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
count: 0
```

`log` captures `count = 0` at the time the callback was created (component mount). The empty dependency array means `log` is never recreated. Every time "Increment" is clicked, the component re-renders with a new `count` value, but `log` still has the stale closure over `count = 0`. Clicking "Log" calls the stale function, which logs `0`.

</details>

---

### Q21

```jsx
function Timer() {
  const [seconds, setSeconds] = React.useState(0);

  const start = React.useCallback(() => {
    const id = setInterval(() => {
      setSeconds(prev => prev + 1); // functional update
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty deps

  React.useEffect(() => {
    const cleanup = start();
    return cleanup;
  }, [start]);

  return <p>{seconds}s</p>;
}
```

#### ❓ Does the timer work correctly? Does it cause any issues?

<details>
<summary>✅ Answer</summary>

```txt
Yes — the timer works correctly and does not cause issues.

Explanation:
  - start has empty deps and is therefore stable across renders.
  - useEffect has [start] as dep — since start never changes, the effect runs only once (on mount).
  - Inside the interval, setSeconds(prev => prev + 1) uses a functional update,
    so it does not need to capture `seconds` from a closure — no stale closure.
  - The cleanup function (clearInterval) is returned from the effect via start(),
    so the interval is properly cleared on unmount.
```

</details>

---

## 6. Edge Cases

### Q22

```jsx
function App() {
  const [count, setCount] = React.useState(0);
  let renderCount = 0;

  const fn = React.useCallback(() => {
    return renderCount;
  }, []);

  renderCount++;

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>inc</button>
      <button onClick={() => console.log(fn())}>log renderCount</button>
    </div>
  );
}
```

#### ❓ After incrementing count 5 times and then clicking "log renderCount", what is logged?

<details>
<summary>✅ Answer</summary>

```txt
0
```

`renderCount` is a regular `let` variable (not state, not a ref). It is reset to `0` at the start of every render call and incremented to `1` within that render. It does not persist between renders. `fn` captures `renderCount = 0` from the render in which it was created (the initial render, since deps is `[]`). The closure over `renderCount` always sees `0`.

</details>

---

### Q23

```jsx
function App() {
  const [count, setCount] = React.useState(0);
  const [, dispatch] = React.useReducer(x => x + 1, 0);

  let prevDispatch = React.useRef(dispatch);
  let prevSetCount = React.useRef(setCount);

  React.useEffect(() => {
    console.log('dispatch stable?', dispatch === prevDispatch.current);
    console.log('setCount stable?', setCount === prevSetCount.current);
    prevDispatch.current = dispatch;
    prevSetCount.current = setCount;
  });

  return <button onClick={() => setCount(c => c + 1)}>inc</button>;
}
```

#### ❓ After clicking the button twice, what is logged on the second render?

<details>
<summary>✅ Answer</summary>

```txt
dispatch stable? true
setCount stable? true
```

React guarantees that `dispatch` from `useReducer` and `setState` from `useState` have stable identities across renders. They are created once and the same reference is reused for the entire lifetime of the component. This is why it is safe to omit them from `useCallback` dependency arrays.

</details>

---

### Q24

```jsx
function useStableCallback(callback) {
  const ref = React.useRef(callback);

  React.useLayoutEffect(() => {
    ref.current = callback;
  });

  return React.useCallback((...args) => {
    return ref.current(...args);
  }, []); // empty deps — permanently stable reference
}

function Counter() {
  const [count, setCount] = React.useState(0);

  const logCount = useStableCallback(() => {
    console.log('count:', count);
  });

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>inc</button>
      <button onClick={logCount}>log</button>
    </div>
  );
}
```

#### ❓ After clicking "inc" 3 times, what does clicking "log" print?

<details>
<summary>✅ Answer</summary>

```txt
count: 3
```

`useStableCallback` is a pattern for a "stable yet always-fresh" callback. The trick:
1. `ref.current` is updated to the latest callback after every render via `useLayoutEffect`.
2. The returned function has an empty dep array — its reference never changes.
3. When invoked, it calls `ref.current(...)` which points to the latest version of the callback.

This gives a permanently stable function reference that always executes the latest closure — no stale closure problem. It is sometimes called the "useEvent" pattern.

</details>

---

### Q25

```jsx
const MemoChild = React.memo(({ onAction }) => {
  console.log('MemoChild rendered');
  return <button onClick={onAction}>Action</button>;
});

function Parent() {
  const [a, setA] = React.useState(0);
  const [b, setB] = React.useState(0);

  const handleAction = React.useCallback(() => {
    console.log('a:', a, 'b:', b);
  }, [a]); // b is intentionally omitted

  return (
    <div>
      <button onClick={() => setA(v => v + 1)}>inc A</button>
      <button onClick={() => setB(v => v + 1)}>inc B</button>
      <MemoChild onAction={handleAction} />
    </div>
  );
}
```

#### ❓ After clicking "inc A" once then "inc B" once, what happens to MemoChild re-renders and what does clicking "Action" log?

<details>
<summary>✅ Answer</summary>

```txt
Re-render count:
  "inc A" → MemoChild RE-RENDERS (a changed → new handleAction reference)
  "inc B" → MemoChild does NOT re-render (a unchanged → same handleAction reference)

Action log after sequence:
  a: 1 b: 0
  (b is 1 in state, but handleAction has stale closure over b = 0 because b is not in deps)
```

Two issues demonstrated:
1. MemoChild only re-renders when `a` changes (because `a` is in deps and drives handleAction's reference stability).
2. Inside `handleAction`, `b` is stale because it is not listed in the dependency array — the eslint `exhaustive-deps` rule would warn about this omission.

</details>

---

## ✅ Topics Covered

| Topic | Questions |
|-------|-----------|
| Reference equality across renders | Q1, Q2, Q3, Q4, Q5 |
| React.memo + useCallback interaction | Q6, Q7, Q8, Q9, Q10 |
| Stale closures in useCallback | Q11, Q12, Q13, Q14 |
| useCallback vs useMemo distinction | Q15, Q16, Q17 |
| Dependency array correctness | Q18, Q19, Q20, Q21 |
| Edge cases (StrictMode, dispatch, custom hooks) | Q22, Q23, Q24, Q25 |
