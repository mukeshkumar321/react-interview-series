## React Performance — Tricky Output Questions

> These questions test your ability to predict render counts, understand memoization boundaries, reason about referential equality, and identify performance pitfalls. Each question reflects a real scenario from senior React interviews.

---

## 1. Re-render Triggers

### Q1

```jsx
const Child = React.memo(function Child({ name }) {
  console.log('Child rendered');
  return <p>{name}</p>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child name="Alice" />
    </>
  );
}
```

#### ❓ How many times does "Child rendered" log after 3 button clicks?

<details>
<summary>✅ Answer</summary>

```txt
Once — only on the initial mount.
```

**Explanation:** `React.memo` wraps `Child`. On every button click, `Parent` re-renders, but before re-rendering `Child`, React shallow-compares the old `name` prop with the new `name` prop. `Object.is("Alice", "Alice")` is `true`. React skips re-rendering `Child` entirely. "Child rendered" only logs on the initial mount.

</details>

---

### Q2

```jsx
const Child = React.memo(function Child({ config }) {
  console.log('Child rendered');
  return <p>{config.theme}</p>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child config={{ theme: 'dark' }} />
    </>
  );
}
```

#### ❓ How many times does "Child rendered" log after 3 button clicks?

<details>
<summary>✅ Answer</summary>

```txt
4 times — once on mount, then once per each of the 3 button clicks.
```

**Explanation:** `{ theme: 'dark' }` is an object literal created inside `Parent`'s render function. Every time `Parent` re-renders, a new object is created with a new reference. `Object.is(oldConfig, newConfig)` is `false` because they are different objects in memory, even though the content is identical. `React.memo`'s shallow comparison sees "different prop" and re-renders `Child`. To fix, define `config` outside the component or use `useMemo`.

</details>

---

### Q3

```jsx
const ThemeContext = createContext('light');

const Button = React.memo(function Button() {
  console.log('Button rendered');
  return <button>Click</button>;
});

function App() {
  const [count, setCount] = useState(0);

  return (
    <ThemeContext.Provider value="dark">
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Button />
    </ThemeContext.Provider>
  );
}
```

#### ❓ `Button` does not consume `ThemeContext`. Does it re-render when `count` changes?

<details>
<summary>✅ Answer</summary>

```txt
No. Button does NOT re-render when count changes.
```

**Explanation:** `React.memo` prevents `Button` from re-rendering because its props have not changed (it receives no props). Context changes only cause re-renders in components that actually *consume* that context via `useContext`. Since `Button` does not call `useContext(ThemeContext)`, a context value change has no effect on it either. `React.memo` + no prop changes = no re-render.

</details>

---

### Q4

```jsx
const ThemeContext = createContext('light');

function ThemedButton() {
  const theme = useContext(ThemeContext);
  console.log('ThemedButton rendered', theme);
  return <button className={theme}>Click</button>;
}

function App() {
  const [count, setCount] = useState(0);
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={() => setTheme('dark')}>Dark mode</button>
      <ThemedButton />
    </ThemeContext.Provider>
  );
}
```

#### ❓ What logs when the count button is clicked 3 times, then dark mode is clicked once?

<details>
<summary>✅ Answer</summary>

```txt
// Initial mount:
ThemedButton rendered light

// Count button clicked 3 times:
ThemedButton rendered light  (3 times — parent re-renders cascade)

// Dark mode clicked:
ThemedButton rendered dark
```

**Explanation:** `ThemedButton` is not wrapped in `React.memo`, so every parent re-render causes it to re-render. Even though the context value hasn't changed on the count clicks, the parent re-render propagates to all children. When the theme changes, the component re-renders again due to both the parent re-render and the context value change (but React deduplicates and renders once).

</details>

---

### Q5

```jsx
function Parent() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  console.log('Parent rendered');

  return (
    <>
      <button onClick={() => setA(0)}>Set A to 0</button>
      <button onClick={() => setB(b + 1)}>Inc B</button>
    </>
  );
}
```

#### ❓ The "Set A to 0" button is clicked when `a` is already `0`. Does "Parent rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
It depends. React MAY call the component function once to confirm the bailout,
but does NOT commit a DOM update. The log may or may not appear.
```

**Explanation:** When `setState` is called with the same value (`Object.is(0, 0) = true`), React performs a bailout optimization. In practice, React may re-render the component once to confirm the bailout, but it does not proceed to commit changes to the DOM and does not re-render children. This one extra render call is an implementation detail. The key point: no visible DOM update occurs, and children do not re-render.

</details>

---

## 2. useMemo and useCallback

### Q6

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');

  const doubled = useMemo(() => {
    console.log('computing doubled');
    return count * 2;
  }, [count]);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={() => setName('Bob')}>Change name</button>
      <p>Doubled: {doubled}</p>
    </>
  );
}
```

#### ❓ "computing doubled" logs on mount. What happens when the "Change name" button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
"computing doubled" does NOT log when the name button is clicked.
```

**Explanation:** `useMemo` recomputes only when a dependency changes. The dependency array is `[count]`. Clicking "Change name" updates `name` state, causing `App` to re-render. But `count` has not changed, so `useMemo` returns the cached `doubled` value without calling the compute function again. This is the purpose of `useMemo` — avoiding recomputation when irrelevant state changes.

</details>

---

### Q7

```jsx
function App() {
  const [items, setItems] = useState([1, 2, 3]);

  const total = useMemo(() => {
    console.log('computing total');
    return items.reduce((sum, n) => sum + n, 0);
  }, [items]);

  function addItem() {
    items.push(4); // mutate
    setItems(items); // same reference
  }

  return (
    <>
      <button onClick={addItem}>Add 4</button>
      <p>Total: {total}</p>
    </>
  );
}
```

#### ❓ After clicking "Add 4", what does "Total" display?

<details>
<summary>✅ Answer</summary>

```txt
Total: 6 — the total does NOT update to 10.
```

**Explanation:** `items.push(4)` mutates the array in place. `setItems(items)` passes the same array reference. `Object.is(oldItems, newItems)` is `true` (same reference). React bails out and does not re-render. Even if React does re-render (the behavior here is implementation-dependent), `useMemo`'s dependency comparison also sees the same reference and skips recomputation. Mutation breaks both React state and `useMemo`.

</details>

---

### Q8

```jsx
const Child = React.memo(function Child({ onClick }) {
  console.log('Child rendered');
  return <button onClick={onClick}>Click me</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []); // empty deps

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Parent count: {count}</button>
      <Child onClick={handleClick} />
    </>
  );
}
```

#### ❓ Does "Child rendered" log when the parent count button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
No. "Child rendered" only logs on the initial mount.
```

**Explanation:** `useCallback` with `[]` creates the `handleClick` function once and returns the same reference on every subsequent render. `React.memo` on `Child` compares `prevProps.onClick` with `nextProps.onClick` using `Object.is`. Since `handleClick` is the same reference, they are equal. `React.memo` skips the re-render. This is the `useCallback + React.memo` optimization pattern working correctly.

</details>

---

### Q9

```jsx
function App() {
  const [count, setCount] = useState(0);

  const getCount = useCallback(() => count, [count]);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(getCount());
    }, 1000);
    return () => clearInterval(id);
  }, []); // missing getCount in deps

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ The button is clicked 3 times in rapid succession. What does the interval log after 1 second?

<details>
<summary>✅ Answer</summary>

```txt
0 — always logs 0.
```

**Explanation:** The effect runs once on mount with `getCount` captured at that point. Even though `useCallback` creates a new `getCount` when `count` changes, the effect never re-runs (missing dep). The `getCount` reference inside the interval closure is the original one from mount, which closed over `count = 0`. This is a stale closure via a stale `useCallback` reference. Fix: add `getCount` to the effect's dependency array.

</details>

---

### Q10

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  const sum = useMemo(() => a + b, [a, b]);

  const handleA = useCallback(() => setA(prev => prev + 1), []);
  const handleB = useCallback(() => setB(prev => prev + 1), []);

  console.log('App rendered');

  return (
    <>
      <button onClick={handleA}>A: {a}</button>
      <button onClick={handleB}>B: {b}</button>
      <p>Sum: {sum}</p>
    </>
  );
}
```

#### ❓ "App rendered" logs on mount. What logs when button A is clicked?

<details>
<summary>✅ Answer</summary>

```txt
App rendered  — logged once when button A is clicked.
```

**Explanation:** Clicking button A calls `setA`, which changes state. `App` re-renders (state update always causes the component to re-render). "App rendered" logs. `sum` recomputes because `a` changed. `handleA` and `handleB` return their cached references (their deps are `[]`). Note: `useMemo` and `useCallback` only prevent *downstream* re-renders of memoized children — they do not prevent the component that holds them from re-rendering.

</details>

---

## 3. React.memo

### Q11

```jsx
const Card = React.memo(function Card({ user }) {
  console.log('Card rendered for', user.name);
  return <div>{user.name}</div>;
});

function App() {
  const [count, setCount] = useState(0);
  const user = useMemo(() => ({ name: 'Alice', id: 1 }), []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Card user={user} />
    </>
  );
}
```

#### ❓ Does "Card rendered for Alice" log on every button click?

<details>
<summary>✅ Answer</summary>

```txt
No. It only logs on the initial mount.
```

**Explanation:** `user` is created with `useMemo(() => ..., [])`. The empty dependency array means the same object reference is returned on every render. `React.memo` compares old and new `user` with `Object.is`. Same reference → `true` → skip re-render. Both `useMemo` (to stabilize the object) and `React.memo` (to prevent re-render) are needed together for this optimization to work.

</details>

---

### Q12

```jsx
const List = React.memo(function List({ items, onDelete }) {
  console.log('List rendered');
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
  const [items, setItems] = useState([
    { id: 1, name: 'Apple' },
    { id: 2, name: 'Banana' },
  ]);

  function handleDelete(id) {
    setItems(prev => prev.filter(item => item.id !== id));
  }

  return <List items={items} onDelete={handleDelete} />;
}
```

#### ❓ The "Delete Apple" button is clicked. Does `React.memo` prevent the List re-render?

<details>
<summary>✅ Answer</summary>

```txt
No. List re-renders twice on deletion:
1. Because items changed (new filtered array reference)
2. Because handleDelete changed (new function reference each render)
```

**Explanation:** After deletion, `setItems` produces a new array → `items` prop is a new reference. Additionally, `handleDelete` is defined inline without `useCallback`, so it's a new function reference every render. Both `items` and `onDelete` are "changed" from `React.memo`'s perspective. Even if `items` changed legitimately, the `onDelete` instability means `React.memo` would still re-render even on unrelated parent state changes. Fix: wrap `handleDelete` with `useCallback`.

</details>

---

### Q13

```jsx
const Item = React.memo(
  function Item({ value, threshold }) {
    console.log('Item rendered');
    return <p>{value}</p>;
  },
  (prevProps, nextProps) => {
    // Return true = skip re-render
    return Math.abs(prevProps.value - nextProps.value) < nextProps.threshold;
  }
);

function App() {
  const [value, setValue] = useState(0);

  return (
    <>
      <button onClick={() => setValue(v => v + 3)}>+3</button>
      <button onClick={() => setValue(v => v + 10)}>+10</button>
      <Item value={value} threshold={5} />
    </>
  );
}
```

#### ❓ Starting from value=0: click "+3" once, then "+10" once. How many times does "Item rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
Initial mount: Item rendered  (1 time)
After "+3": NOT rendered (|3 - 0| = 3 < 5 → custom compare returns true → skip)
After "+10": Item rendered  (|13 - 3| = 10 >= 5 → custom compare returns false → re-render)
```

**Explanation:** The custom comparator returns `true` (skip re-render) when the value change is less than the threshold. After "+3", `|3 - 0| = 3 < 5` → skip. After "+10" from base value 3, `|13 - 3| = 10 >= 5` → re-render. Note: the custom comparator receives `prevProps.value = 3` (last rendered value) and `nextProps.value = 13`.

</details>

---

### Q14

```jsx
const Child = React.memo(function Child({ data }) {
  console.log('Child rendered');
  return <p>{data.length}</p>;
});

function App() {
  const [toggle, setToggle] = useState(false);
  const data = [1, 2, 3]; // defined inside component

  return (
    <>
      <button onClick={() => setToggle(t => !t)}>Toggle</button>
      <Child data={data} />
    </>
  );
}
```

#### ❓ Does `React.memo` prevent `Child` from re-rendering on each toggle click?

<details>
<summary>✅ Answer</summary>

```txt
No. Child re-renders on every toggle click.
```

**Explanation:** `data` is an array literal `[1, 2, 3]` declared inside the `App` function body. Every render creates a new array with a new reference. `React.memo` does a shallow comparison: `Object.is(oldData, newData)` is `false` for different array references. `React.memo` sees a changed prop and re-renders `Child`. Fix: move `data` outside the component (if static) or wrap it in `useMemo`.

</details>

---

### Q15

```jsx
const Button = React.memo(function Button({ label, onClick }) {
  console.log('Button rendered:', label);
  return <button onClick={onClick}>{label}</button>;
});

function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>
      <Button label="Increment" onClick={() => setCount(c => c + 1)} />
      <Button label="Reset" onClick={() => setCount(0)} />
    </div>
  );
}
```

#### ❓ What logs after clicking "Increment" once?

<details>
<summary>✅ Answer</summary>

```txt
Button rendered: Increment
Button rendered: Reset
```

**Explanation:** Both buttons re-render even though their labels didn't change. The `onClick` props are arrow functions defined inline in `App`'s render body. Every render creates two new function references. `React.memo` compares `prevProps.onClick` with `nextProps.onClick` — different references → re-render both buttons. Clicking Increment changes `count`, re-renders `App`, creates new function references for both buttons, so both re-render. Fix: `useCallback` for both handlers.

</details>

---

## 4. Concurrent Features

### Q16

```jsx
function App() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    const value = e.target.value;
    setQuery(value); // urgent
    startTransition(() => {
      setResults(computeHeavyResults(value)); // deferred
    });
  }

  console.log('App rendered, isPending:', isPending);

  return <input value={query} onChange={handleChange} />;
}
```

#### ❓ The user types "a". How many times does "App rendered" log and what are the `isPending` values?

<details>
<summary>✅ Answer</summary>

```txt
At minimum 2 renders:
1. App rendered, isPending: true   (urgent query update applied, transition pending)
2. App rendered, isPending: false  (transition completed, results applied)
```

**Explanation:** Typing triggers `setQuery` (urgent) and `startTransition` (deferred). React processes the urgent update first, making `isPending: true` and showing the updated input. The deferred update processes when React has capacity. When it completes, `isPending` becomes `false`. React may batch or split these differently depending on workload, but `isPending: true` always appears before `isPending: false`.

</details>

---

### Q17

```jsx
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    const computed = computeExpensive(query);
    setResults(computed);
  }, [query]);

  return <ul>{results.map(r => <li key={r}>{r}</li>)}</ul>;
}

function App() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults query={deferredQuery} />
    </>
  );
}
```

#### ❓ What is the difference between using `deferredQuery` vs `query` as the prop to `SearchResults`?

<details>
<summary>✅ Answer</summary>

```txt
With deferredQuery:
- input updates immediately (query state)
- SearchResults receives the deferred value which lags behind
- React can interrupt SearchResults re-render to keep input responsive
- User sees instant typing feedback even if SearchResults is slow

With query (direct):
- Both input and SearchResults update in the same render
- If computeExpensive is slow, the input may feel sluggish
```

**Explanation:** `useDeferredValue` creates a value that "lags behind" the current value. React schedules the `SearchResults` re-render at lower priority. If the user types faster than `SearchResults` can render, React discards the intermediate deferred values and jumps to the latest. The input component always renders at high priority with the current `query` value, maintaining responsiveness.

</details>

---

### Q18

```jsx
function SlowComponent({ value }) {
  // Simulate slow render
  const start = Date.now();
  while (Date.now() - start < 100) {} // 100ms blocking

  return <p>{value}</p>;
}

function App() {
  const [input, setInput] = useState('');
  const deferred = useDeferredValue(input);

  return (
    <>
      <input value={input} onChange={e => setInput(e.target.value)} />
      <SlowComponent value={deferred} />
    </>
  );
}
```

#### ❓ Does `useDeferredValue` solve the input lag caused by `SlowComponent`'s 100ms blocking loop?

<details>
<summary>✅ Answer</summary>

```txt
Partially. useDeferredValue allows React to deprioritize SlowComponent updates,
but the synchronous while-loop blocks the main thread unconditionally.
The input will still lag when SlowComponent actually renders.
```

**Explanation:** `useDeferredValue` tells React to delay re-rendering `SlowComponent` until the browser is idle. This means React renders the input update immediately and defers `SlowComponent`. However, once React *does* render `SlowComponent`, the 100ms `while` loop runs synchronously on the main thread — nothing can interrupt it. True concurrent rendering requires components to be interruptible, which synchronous CPU-intensive code prevents. The fix is to move the slow computation to a Web Worker or use `useMemo` to avoid repeating it.

</details>

---

### Q19

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [isPending, startTransition] = useTransition();

  function handleClick() {
    startTransition(() => {
      setCount(c => c + 1);
    });
  }

  return (
    <>
      <button onClick={handleClick}>Count: {count}</button>
      <p>Pending: {String(isPending)}</p>
    </>
  );
}
```

#### ❓ Is the count update instant? When does `isPending` flip back to `false`?

<details>
<summary>✅ Answer</summary>

```txt
The count update is deferred (lower priority). isPending becomes true immediately
and flips back to false once the transition completes and count updates.
```

**Explanation:** `startTransition` marks the `setCount` update as a transition (low priority). React applies the count update when it has capacity. `isPending` is `true` while the transition is pending, allowing the UI to show a loading indicator. For a simple counter, the transition completes almost immediately. `isPending` is most useful when the transition involves slow-to-render components, where the delay is perceptible.

</details>

---

### Q20

```jsx
function App() {
  const [data, setData] = useState(null);

  return (
    <Suspense fallback={<p>Loading...</p>}>
      <DataComponent />
    </Suspense>
  );
}

function DataComponent() {
  const data = use(fetchPromise); // React 19 use() hook
  return <p>{data.name}</p>;
}
```

#### ❓ While `fetchPromise` is pending, what does the user see?

<details>
<summary>✅ Answer</summary>

```txt
The user sees "Loading..." from the Suspense fallback.
```

**Explanation:** When a component throws a Promise (which is what `use()` does internally when the promise is pending), React catches it and renders the nearest `Suspense` boundary's fallback. Once the promise resolves, React re-renders `DataComponent` with the resolved value and replaces the fallback with the actual content. This is the core mechanism of Suspense — declarative loading states without manual `isLoading` checks.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function List({ items }) {
  return (
    <ul>
      {items.map(item => (
        <Item key={Math.random()} item={item} />
      ))}
    </ul>
  );
}

const Item = React.memo(function Item({ item }) {
  console.log('Item rendered:', item.name);
  return <li>{item.name}</li>;
});
```

#### ❓ The parent re-renders with the same `items` array. Does `React.memo` prevent Item re-renders?

<details>
<summary>✅ Answer</summary>

```txt
No. Every Item re-renders AND gets fully remounted on every parent render.
```

**Explanation:** `Math.random()` as a key generates a different value on every render. React uses keys to match old and new elements. When keys change, React cannot reconcile old elements with new ones — it unmounts all old Items and mounts all new ones from scratch. This completely defeats `React.memo` (new elements are always fresh mounts). Keys must be stable, unique identifiers (like `item.id`).

</details>

---

### Q22

```jsx
function ExpensiveList({ items }) {
  const [filter, setFilter] = useState('');

  const filtered = useMemo(
    () => items.filter(item => item.name.includes(filter)),
    [items, filter]
  );

  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      <ul>{filtered.map(item => <li key={item.id}>{item.name}</li>)}</ul>
    </>
  );
}
```

#### ❓ The `items` prop is `[...serverData]` (new array reference on every parent render). Does `useMemo` recompute on every parent render?

<details>
<summary>✅ Answer</summary>

```txt
Yes. useMemo recomputes on every parent render because items is a new
array reference each time.
```

**Explanation:** `useMemo` uses `Object.is` for dependency comparison. `Object.is([...serverData], [...serverData])` is `false` — different array references. Even if the array *contents* are identical, a new spread creates a new reference. `useMemo` always recomputes when it sees a "different" dependency. Fix: memoize `serverData` in the parent, or pass `serverData` directly without spreading.

</details>

---

### Q23

```jsx
function HeavyComponent() {
  const [data] = useState(() => {
    console.log('initializing state');
    return computeHeavy(); // takes 200ms
  });

  console.log('rendering');
  return <p>{data.value}</p>;
}
```

#### ❓ How many times does "initializing state" log across 5 re-renders caused by parent updates?

<details>
<summary>✅ Answer</summary>

```txt
Once — only on the initial mount.
```

**Explanation:** The lazy initializer function passed to `useState` runs exactly once: during the component's initial mount. On every subsequent render, React ignores the initializer argument entirely and reads the stored state value from the fiber. This is the purpose of lazy initialization — avoid re-running expensive computations on re-renders. "rendering" would log 6 times (mount + 5 re-renders).

</details>

---

### Q24

```jsx
function App() {
  const [showList, setShowList] = useState(true);

  return (
    <>
      <button onClick={() => setShowList(s => !s)}>Toggle</button>
      {showList && <VirtualList />}
    </>
  );
}

function VirtualList() {
  const items = Array.from({ length: 10000 }, (_, i) => ({ id: i, name: `Item ${i}` }));

  return (
    <FixedSizeList height={400} itemCount={items.length} itemSize={35}>
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

#### ❓ How many DOM nodes does `FixedSizeList` create for 10,000 items?

<details>
<summary>✅ Answer</summary>

```txt
Approximately 12-20 DOM nodes — only the items visible in the 400px container
plus a small overscan buffer.
```

**Explanation:** `FixedSizeList` from `react-window` virtualizes the list. With `itemSize: 35` and `height: 400`, only `Math.ceil(400 / 35) = 12` items fit in view. react-window renders only those items plus a small overscan buffer (~2 extra items above and below). The total DOM count is around 12-16, not 10,000. This is the core value of windowing/virtualization.

</details>

---

### Q25

```jsx
const UserContext = createContext(null);

function UserProvider({ children }) {
  const [user, setUser] = useState({ name: 'Alice', role: 'admin' });

  const value = { user, setUser };  // new object every render

  return (
    <UserContext.Provider value={value}>
      {children}
    </UserContext.Provider>
  );
}

const Header = React.memo(function Header() {
  const { user } = useContext(UserContext);
  console.log('Header rendered');
  return <h1>Hello {user.name}</h1>;
});

const Sidebar = React.memo(function Sidebar() {
  const { user } = useContext(UserContext);
  console.log('Sidebar rendered');
  return <nav>{user.role}</nav>;
});
```

#### ❓ `UserProvider`'s parent re-renders (unrelated to user state). What happens to Header and Sidebar?

<details>
<summary>✅ Answer</summary>

```txt
Both Header and Sidebar re-render despite React.memo.
```

**Explanation:** `const value = { user, setUser }` creates a new object on every render of `UserProvider`. When `UserProvider`'s parent re-renders, `UserProvider` re-renders, creating a new `value` object. React detects the context value has changed (new reference) and forces all context consumers to re-render — bypassing `React.memo`. Fix: memoize the context value with `useMemo`:

```jsx
const value = useMemo(() => ({ user, setUser }), [user]);
```

Or better: split into `UserStateContext` and `UserDispatchContext` so components only subscribe to what they need.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Re-render Triggers | Q1 – Q5 | React.memo with stable props, new object props, context without useContext, context with useContext, setState with same value |
| useMemo / useCallback | Q6 – Q10 | Dependency-scoped memoization, mutation breaking useMemo, useCallback + React.memo, stale useCallback in effect, render vs downstream optimization |
| React.memo | Q11 – Q15 | useMemo stabilizing objects, handleDelete without useCallback, custom comparator, array reference instability, inline function handlers |
| Concurrent Features | Q16 – Q20 | useTransition isPending renders, useDeferredValue lag, blocking code vs deferred, startTransition priority, Suspense fallback |
| Edge Cases | Q21 – Q25 | Math.random() key defeating memo, spread creating new reference, lazy init runs once, virtualization DOM count, context value object instability |
