## Custom Hooks — Tricky Output Questions

> These questions test your understanding of state isolation, hook composition, useFetch patterns, race conditions, useLocalStorage, and edge cases like calling hooks conditionally or outside components. Each question reflects real scenarios from senior React interviews.

---

## 1. State Isolation

### Q1

```jsx
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  return { count, increment };
}

function ComponentA() {
  const { count, increment } = useCounter(0);
  console.log('A:', count);
  return <button onClick={increment}>A: {count}</button>;
}

function ComponentB() {
  const { count, increment } = useCounter(0);
  console.log('B:', count);
  return <button onClick={increment}>B: {count}</button>;
}

function App() {
  return <><ComponentA /><ComponentB /></>;
}
```

#### ❓ ComponentA's button is clicked 3 times. What do A and B display? What does "B" log?

<details>
<summary>✅ Answer</summary>

```txt
A displays: 3
B displays: 0
"B" never logs again after initial render (B's state doesn't change).
```

**Explanation:** Each call to `useCounter` creates an independent `useState` instance. `ComponentA` and `ComponentB` each have their own `count` state. Clicking ComponentA's button only updates ComponentA's state. ComponentB's state remains at `0`. Custom hooks reuse logic, not state. The state lives in the component instance that called the hook, not in the hook function itself.

</details>

---

### Q2

```jsx
function useFlag() {
  const [flag, setFlag] = useState(false);
  return { flag, toggle: () => setFlag(f => !f) };
}

function Parent() {
  const { flag: flagA, toggle: toggleA } = useFlag();
  const { flag: flagB, toggle: toggleB } = useFlag();

  console.log('flagA:', flagA, 'flagB:', flagB);

  return (
    <>
      <button onClick={toggleA}>Toggle A</button>
      <button onClick={toggleB}>Toggle B</button>
    </>
  );
}
```

#### ❓ "Toggle A" is clicked twice. What is logged each time?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
flagA: false flagB: false

// After 1st click:
flagA: true flagB: false

// After 2nd click:
flagA: false flagB: false
```

**Explanation:** Even though both calls use `useFlag`, they are separate hook invocations in the same component (`Parent`). Each call maintains independent state. `useFlag()` called a second time creates a second independent `useState(false)`. Toggling A twice: `false → true → false`. B stays at `false` throughout.

</details>

---

### Q3

```jsx
let globalCount = 0;

function useCounter() {
  const [count, setCount] = useState(() => {
    globalCount++;
    return 0;
  });
  return { count, increment: () => setCount(c => c + 1) };
}

function ComponentA() {
  const { count } = useCounter();
  return <p>A: {count}</p>;
}

function ComponentB() {
  const { count } = useCounter();
  return <p>B: {count}</p>;
}
```

#### ❓ After both components mount (in production), what is `globalCount`?

<details>
<summary>✅ Answer</summary>

```txt
globalCount: 2
```

**Explanation:** Each component instance calls `useCounter`, which calls `useState` with a lazy initializer. The lazy initializer runs once per component instance. Two components = two `useState` initializations = two `globalCount++` calls. In production, `globalCount = 2`. In Strict Mode development, React double-invokes initializers, so `globalCount = 4`. This illustrates that state is per component instance — each instance runs its own initialization.

</details>

---

### Q4

```jsx
function useSharedResource() {
  const [value, setValue] = useState('shared?');
  return [value, setValue];
}

function Reader() {
  const [value] = useSharedResource();
  return <p>Reader: {value}</p>;
}

function Writer() {
  const [, setValue] = useSharedResource();
  return <button onClick={() => setValue('updated')}>Update</button>;
}
```

#### ❓ After clicking "Update", does `Reader` show "updated"?

<details>
<summary>✅ Answer</summary>

```txt
No — Reader continues to show "shared?".
```

**Explanation:** `useSharedResource` is called independently in `Reader` and `Writer`. Each call creates its own `useState` instance. When `Writer` calls `setValue('updated')`, it updates `Writer`'s own state — not `Reader`'s. `Reader` has a completely separate state that stays at `'shared?'`. To share state between components, you must lift the state to a common ancestor or use Context. Custom hooks are logic-sharing mechanisms, not state-sharing mechanisms.

</details>

---

### Q5

```jsx
function useStepCounter(step = 1) {
  const [count, setCount] = useState(0);
  const increment = useCallback(() => {
    setCount(c => c + step);
  }, [step]);
  return { count, increment };
}

function ByOne() {
  const { count, increment } = useStepCounter(1);
  return <button onClick={increment}>By1: {count}</button>;
}

function ByFive() {
  const { count, increment } = useStepCounter(5);
  return <button onClick={increment}>By5: {count}</button>;
}
```

#### ❓ "By1" is clicked twice, "By5" is clicked once. What do each display?

<details>
<summary>✅ Answer</summary>

```txt
ByOne: 2
ByFive: 5
```

**Explanation:** `useStepCounter(1)` in `ByOne` creates a counter with `step = 1`. `useStepCounter(5)` in `ByFive` creates a separate counter with `step = 5`. Each has independent state. After 2 clicks on ByOne: `0 + 1 + 1 = 2`. After 1 click on ByFive: `0 + 5 = 5`. The custom hook accepts parameters to configure its behavior — a key pattern for flexible custom hooks.

</details>

---

## 2. Custom Hook Composition

### Q6

```jsx
function useBoolean(initial = false) {
  const [value, setValue] = useState(initial);
  return {
    value,
    setTrue: () => setValue(true),
    setFalse: () => setValue(false),
    toggle: () => setValue(v => !v),
  };
}

function useModal() {
  const { value: isOpen, setTrue: open, setFalse: close } = useBoolean(false);
  return { isOpen, open, close };
}

function App() {
  const { isOpen, open, close } = useModal();

  console.log('isOpen:', isOpen);

  return (
    <>
      <button onClick={open}>Open</button>
      <button onClick={close}>Close</button>
    </>
  );
}
```

#### ❓ "Open" is clicked, then "Close". What sequence does `console.log` produce?

<details>
<summary>✅ Answer</summary>

```txt
isOpen: false   ← initial render
isOpen: true    ← after "Open"
isOpen: false   ← after "Close"
```

**Explanation:** `useModal` is composed from `useBoolean`. Clicking "Open" calls the `setTrue` function from `useBoolean`, which calls `setValue(true)` — updating `isOpen` to `true` and triggering a re-render. Clicking "Close" calls `setFalse` → `setValue(false)` → `isOpen` becomes `false`. The composition chain works correctly. `useModal` adds semantic naming (`open`, `close`) on top of `useBoolean`'s generic API.

</details>

---

### Q7

```jsx
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  return { count, increment: () => setCount(c => c + 1) };
}

function useDoubleCounter(initial = 0) {
  const counterA = useCounter(initial);
  const counterB = useCounter(initial);
  return {
    a: counterA.count,
    b: counterB.count,
    incrementA: counterA.increment,
    incrementB: counterB.increment,
    total: counterA.count + counterB.count,
  };
}

function App() {
  const { a, b, incrementA, total } = useDoubleCounter(0);

  return (
    <>
      <p>A: {a}, B: {b}, Total: {total}</p>
      <button onClick={incrementA}>Inc A</button>
    </>
  );
}
```

#### ❓ After clicking "Inc A" 3 times, what is displayed?

<details>
<summary>✅ Answer</summary>

```txt
A: 3, B: 0, Total: 3
```

**Explanation:** `useDoubleCounter` composes two separate `useCounter` calls. Each creates independent state. `incrementA` only updates the first counter's state. `total` is derived by summing both counts: `3 + 0 = 3`. This demonstrates hook composition — `useDoubleCounter` builds on `useCounter` to provide a higher-level abstraction with two independent counters and a derived total.

</details>

---

### Q8

```jsx
function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debounced;
}

function useSearch(query) {
  const debouncedQuery = useDebounce(query, 500);
  const [results, setResults] = useState([]);

  useEffect(() => {
    if (!debouncedQuery) {
      setResults([]);
      return;
    }
    console.log('searching for:', debouncedQuery);
    // simulate search
    setResults([debouncedQuery + '_result']);
  }, [debouncedQuery]);

  return results;
}

function App() {
  const [query, setQuery] = useState('');
  const results = useSearch(query);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>{results.map((r, i) => <li key={i}>{r}</li>)}</ul>
    </>
  );
}
```

#### ❓ The user types "r", "re", "rea", "reac", "react" each 100ms apart (faster than the 500ms debounce). How many times does "searching for:" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time: "searching for: react"
```

**Explanation:** `useDebounce` resets the timer on every value change. Each new character typed cancels the previous timeout and starts a 500ms timer. Since each keystroke is only 100ms apart, the timer is always cancelled before firing. Only after the user finishes typing "react" and 500ms pass does the timer fire, updating `debouncedQuery` to `"react"`. The `useSearch` effect then runs once. This is the core behavior of debouncing — reducing frequent calls to one.

</details>

---

### Q9

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const stored = localStorage.getItem(key);
      return stored ? JSON.parse(stored) : initial;
    } catch {
      return initial;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function useTheme() {
  return useLocalStorage('theme', 'light');
}

function App() {
  const [theme, setTheme] = useTheme();
  console.log('theme:', theme);
  return <button onClick={() => setTheme('dark')}>Go Dark</button>;
}
```

#### ❓ After clicking "Go Dark", what is logged and what is stored in localStorage?

<details>
<summary>✅ Answer</summary>

```txt
// Logs:
theme: light   ← initial render
theme: dark    ← after button click

// localStorage after click:
localStorage.getItem('theme') === '"dark"'
```

**Explanation:** `useTheme` composes `useLocalStorage` with a specific key and default. Clicking "Go Dark" calls `setTheme('dark')` → updates state to `'dark'` → re-render logs `'dark'` → the `useEffect` in `useLocalStorage` fires and persists `JSON.stringify('dark')` = `'"dark"'` to localStorage. The value is double-serialized: once by `JSON.stringify` before storing. On next app load, `JSON.parse('"dark"')` gives back `'dark'`.

</details>

---

### Q10

```jsx
function usePrevious(value) {
  const ref = useRef(undefined);
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

function useValueWithHistory(initial) {
  const [value, setValue] = useState(initial);
  const previous = usePrevious(value);
  return { value, previous, setValue };
}

function App() {
  const { value, previous, setValue } = useValueWithHistory(0);

  return (
    <>
      <p>current: {value}, previous: {previous ?? 'none'}</p>
      <button onClick={() => setValue(v => v + 1)}>+</button>
    </>
  );
}
```

#### ❓ What is displayed initially, after the 1st click, and after the 2nd click?

<details>
<summary>✅ Answer</summary>

```txt
// Initial:
current: 0, previous: none

// After 1st click:
current: 1, previous: 0

// After 2nd click:
current: 2, previous: 1
```

**Explanation:** `useValueWithHistory` composes `usePrevious` into a higher-level hook. `usePrevious` updates `ref.current` after each render. During render N, `ref.current` contains render N-1's value. So `previous` always shows the value from one render ago — correctly tracking history. The `?? 'none'` fallback handles the initial `undefined` case gracefully.

</details>

---

## 3. useFetch Pattern

### Q11

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch(url)
      .then(r => r.json())
      .then(d => {
        setData(d);
        setLoading(false);
      });
  }, [url]);

  return { data, loading };
}

function Profile({ userId }) {
  const { data, loading } = useFetch(`/api/users/${userId}`);

  if (loading) return <p>Loading...</p>;
  return <p>{data?.name}</p>;
}

function App() {
  const [userId, setUserId] = useState(1);

  return (
    <>
      <Profile userId={userId} />
      <button onClick={() => setUserId(2)}>Load User 2</button>
    </>
  );
}
```

#### ❓ When "Load User 2" is clicked, what is the problem with this `useFetch` implementation?

<details>
<summary>✅ Answer</summary>

```txt
Race condition: if the response for userId=1 arrives after the response for userId=2,
the Profile will display data for userId=1 (stale data).

Also: when the URL changes, `loading` is not reset to `true`, so there's a flash
of the old data before the new data arrives.
```

**Explanation:** When `userId` changes, the effect re-runs and starts a new fetch. But the old fetch (for userId=1) may still be in-flight. If it resolves last, `setData(staleData)` overwrites the fresh data from userId=2. Fix: use an `ignore` flag in the cleanup, and reset `loading` to `true` at the start of each effect run. Without cleanup, this is also a memory leak risk.

</details>

---

### Q12

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;

    let ignore = false;
    setLoading(true);
    setData(null);
    setError(null);

    fetch(url)
      .then(r => r.json())
      .then(d => {
        if (!ignore) {
          setData(d);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!ignore) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => { ignore = true; };
  }, [url]);

  return { data, loading, error };
}
```

#### ❓ What does the `ignore` flag prevent? What happens to a pending fetch when the component unmounts?

<details>
<summary>✅ Answer</summary>

```txt
The `ignore` flag prevents stale responses from updating state after:
1. The URL changed (new fetch started, old one should be ignored)
2. The component unmounted (should not setState on unmounted component)

When the component unmounts, the cleanup sets `ignore = true`.
The fetch may still complete in the background, but `setData` and `setError`
are not called because `if (!ignore)` is false.
```

**Explanation:** This is the correct race condition fix. The `ignore` flag is a closure variable captured by the fetch callbacks. When the effect cleans up (URL change or unmount), `ignore` is set to `true`. Any pending callbacks from the previous fetch see `ignore = true` and skip the state updates. This prevents both race conditions and "setState on unmounted component" warnings.

</details>

---

### Q13

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!url) return;

    const controller = new AbortController();
    setLoading(true);

    fetch(url, { signal: controller.signal })
      .then(r => r.json())
      .then(d => {
        setData(d);
        setLoading(false);
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setLoading(false);
        }
      });

    return () => controller.abort();
  }, [url]);

  return { data, loading };
}
```

#### ❓ The component using `useFetch` unmounts while a fetch is in-flight. What happens?

<details>
<summary>✅ Answer</summary>

```txt
The cleanup function calls controller.abort().
The in-flight fetch is cancelled at the network level.
The .catch handler receives an AbortError, which is ignored by the if-guard.
No state updates are performed after unmount.
```

**Explanation:** `AbortController` provides actual network-level cancellation (supported in modern browsers). The cleanup aborts the fetch, which causes the promise to reject with an `AbortError`. The catch handler checks `err.name !== 'AbortError'` and skips `setLoading(false)` for aborted requests. This is cleaner than the `ignore` flag pattern because it also cancels the actual HTTP request, freeing network resources.

</details>

---

### Q14

```jsx
function useFetch(url) {
  const [state, setState] = useState({ data: null, loading: true, error: null });

  useEffect(() => {
    let ignore = false;

    setState({ data: null, loading: true, error: null });

    fetch(url)
      .then(r => r.json())
      .then(data => {
        if (!ignore) setState({ data, loading: false, error: null });
      })
      .catch(error => {
        if (!ignore) setState({ data: null, loading: false, error: error.message });
      });

    return () => { ignore = true; };
  }, [url]);

  return state;
}

function App() {
  const [url, setUrl] = useState('/api/users/1');
  const { data, loading, error } = useFetch(url);

  console.log('loading:', loading, 'data:', data?.id);

  return (
    <button onClick={() => setUrl('/api/users/2')}>Load User 2</button>
  );
}
```

#### ❓ After clicking "Load User 2", what is logged before and after the fetch completes?

<details>
<summary>✅ Answer</summary>

```txt
// Before fetch (immediately after click):
loading: true  data: undefined

// After fetch completes:
loading: false  data: 2
```

**Explanation:** When the URL changes, the effect re-runs. It immediately calls `setState({ data: null, loading: true, error: null })`, triggering a re-render that shows `loading: true, data: undefined`. After the fetch resolves and `ignore` is still false, `setState({ data: user2, loading: false, error: null })` is called, triggering another re-render with the new data. Using a single state object for `{ data, loading, error }` ensures they update atomically.

</details>

---

### Q15

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, [url]);

  return data;
}

function App() {
  const user = useFetch('/api/user');
  const posts = useFetch('/api/posts');

  console.log('user:', user?.name, 'posts:', posts?.length);

  return <p>{user?.name}: {posts?.length} posts</p>;
}
```

#### ❓ What is logged on initial render and after both fetches complete (assuming user loads first)?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
user: undefined  posts: undefined

// After user fetch completes:
user: Alice  posts: undefined

// After posts fetch completes:
user: Alice  posts: 5
```

**Explanation:** Both `useFetch` calls run concurrently — they are separate hook instances with separate effects. On initial render, both states are `null`. As each fetch resolves, it updates its respective state, triggering a re-render. The component renders three times: initial, after user loads, after posts load. Using multiple custom hook calls for parallel data fetching is idiomatic — each hook instance is independent.

</details>

---

## 4. useLocalStorage

### Q16

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored !== null ? JSON.parse(stored) : initial;
  });

  const setWithStorage = (newValue) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, setWithStorage];
}

function App() {
  const [count, setCount] = useLocalStorage('count', 0);

  return (
    <button onClick={() => setCount(count + 1)}>Count: {count}</button>
  );
}
```

#### ❓ The page is refreshed after clicking 5 times. What does the button show after refresh?

<details>
<summary>✅ Answer</summary>

```txt
Count: 5
```

**Explanation:** Each click calls `setWithStorage(count + 1)`, which both updates state and writes to `localStorage`. After 5 clicks, `localStorage.getItem('count')` contains `"5"`. On page refresh, the component remounts. The `useState` lazy initializer reads `localStorage.getItem('count')` → `"5"` → `JSON.parse("5")` → `5`. The button initializes with `5` from the persisted storage.

</details>

---

### Q17

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored !== null ? JSON.parse(stored) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function Counter() {
  const [count, setCount] = useLocalStorage('counter', 0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ `Counter` is used in two places in the app. If both components are visible, do they stay in sync when one is clicked?

<details>
<summary>✅ Answer</summary>

```txt
No — they do NOT stay in sync in real time.
```

**Explanation:** Each `useLocalStorage` call creates independent state. When one counter's button is clicked, its state updates and writes to localStorage. The other counter component has its own state that is NOT watching localStorage for changes. Both read from localStorage on mount, but after that, they diverge. To sync multiple components to the same localStorage value, you would need to listen for the `storage` event (which fires when localStorage changes from another tab) or use a shared state (Context/zustand/etc).

</details>

---

### Q18

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initial;
    } catch (e) {
      return initial;
    }
  });

  const set = useCallback((val) => {
    try {
      const toStore = val instanceof Function ? val(value) : val;
      setValue(toStore);
      window.localStorage.setItem(key, JSON.stringify(toStore));
    } catch (e) {
      // silent fail
    }
  }, [key, value]);

  return [value, set];
}
```

#### ❓ The `set` function accepts a function like `setState`. What is the problem when `set(prev => prev + 1)` is called rapidly (e.g., in a loop)?

<details>
<summary>✅ Answer</summary>

```txt
The `value` in the useCallback closure is stale — `val instanceof Function ? val(value) : val`
uses the `value` from when the callback was created, not the latest state.
```

**Explanation:** `useCallback` has `[key, value]` as deps — it is recreated when `value` changes. However, if `set(prev => prev + 1)` is called rapidly (multiple times before state updates propagate), each call uses the `value` from the render that created the callback — a stale snapshot. The fix: use `setValue` with a functional updater to get the latest state from React's queue, and persist to localStorage after the setState has processed, ideally in a separate `useEffect`.

</details>

---

### Q19

```jsx
function useSSRSafeLocalStorage(key, initial) {
  const [value, setValue] = useState(initial); // always initial on first render

  useEffect(() => {
    // Runs only on client, after mount
    const stored = localStorage.getItem(key);
    if (stored !== null) {
      setValue(JSON.parse(stored));
    }
  }, [key]);

  const set = (newValue) => {
    setValue(newValue);
    localStorage.setItem(key, JSON.stringify(newValue));
  };

  return [value, set];
}

function App() {
  const [theme, setTheme] = useSSRSafeLocalStorage('theme', 'light');
  console.log('theme:', theme);
  return <button onClick={() => setTheme('dark')}>Dark</button>;
}
```

#### ❓ What is the sequence of `theme` logs on initial load (assuming localStorage has 'dark' stored)?

<details>
<summary>✅ Answer</summary>

```txt
theme: light    ← initial render (always uses `initial` to avoid hydration mismatch)
theme: dark     ← after mount effect reads localStorage and updates state
```

**Explanation:** This hook is designed to be SSR-safe. During SSR (server-side render), `localStorage` doesn't exist. By initializing with `initial` and reading localStorage only inside `useEffect` (which runs client-side only), hydration mismatches are avoided — the server and initial client render both show `'light'`. After the client mounts, the effect reads the stored value (`'dark'`) and updates state. This causes a brief flash of `'light'` before `'dark'` — a known trade-off for SSR compatibility.

</details>

---

### Q20

```jsx
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    const stored = localStorage.getItem(key);
    return stored !== null ? JSON.parse(stored) : initial;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

function App() {
  const [a, setA] = useLocalStorage('item', 'alpha');
  const [b, setB] = useLocalStorage('item', 'beta'); // same key!

  return (
    <>
      <p>a: {a}, b: {b}</p>
      <button onClick={() => setA('changed')}>Change A</button>
    </>
  );
}
```

#### ❓ Both hooks use the key `'item'`. After clicking "Change A", what do `a` and `b` display?

<details>
<summary>✅ Answer</summary>

```txt
a: changed, b: alpha
```

**Explanation:** Both hooks use the same localStorage key but have independent state. Initially, the first hook reads localStorage (returns `'alpha'` or whatever is stored) and the second reads the same value. After clicking "Change A": `setA('changed')` updates `a`'s state to `'changed'` and writes `'changed'` to localStorage. But `b` has its own independent state that is not informed of this localStorage write. `b` still shows `'alpha'` (its own state). localStorage is now `'changed'` — but `b` won't see that until it re-reads from storage. Using the same key across multiple hook instances causes inconsistency.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function useConditionalHook(enabled) {
  if (!enabled) {
    return null;
  }
  const [value, setValue] = useState(0); // conditional hook call!
  return value;
}

function App() {
  const [enabled, setEnabled] = useState(true);
  const value = useConditionalHook(enabled);

  return (
    <>
      <p>{value}</p>
      <button onClick={() => setEnabled(false)}>Disable</button>
    </>
  );
}
```

#### ❓ What happens when "Disable" is clicked?

<details>
<summary>✅ Answer</summary>

```txt
React throws an error:
"React has detected a change in the order of Hooks called by App."
"Rendered fewer hooks than expected. This may be caused by an accidental early return statement."
```

**Explanation:** The Rules of Hooks prohibit calling hooks conditionally. On the first render, `useConditionalHook(true)` calls `useState(0)` — hook slot 1 is used. When "Disable" is clicked and the hook is called with `false`, the early return is hit before `useState`. React expects to find a hook at slot 1 but finds none — the hook count changed. React throws an error about the hook order violation.

</details>

---

### Q22

```jsx
function useCounter() {
  const [count, setCount] = useState(0);
  return [count, () => setCount(c => c + 1)];
}

// Attempting to use a custom hook outside a component
const [count, increment] = useCounter(); // called outside a React component

console.log(count);
```

#### ❓ What happens when this code runs?

<details>
<summary>✅ Answer</summary>

```txt
React throws an error:
"Invalid hook call. Hooks can only be called inside of the body of a function component."
```

**Explanation:** Hooks (including custom hooks) can only be called inside React functional components or other custom hooks. Calling `useCounter()` at the module level (outside any component) violates this rule. React requires a component instance context to manage hook state. Without it, React has nowhere to store the `useState` state. React throws an invariant violation error at runtime.

</details>

---

### Q23

```jsx
class ClassComponent extends React.Component {
  render() {
    // Attempting to use a hook in a class component
    const [count, setCount] = useState(0);
    return <p>{count}</p>;
  }
}
```

#### ❓ What happens when this class component renders?

<details>
<summary>✅ Answer</summary>

```txt
React throws an error:
"Invalid hook call. Hooks can only be called inside of the body of a function component."
```

**Explanation:** Hooks cannot be used in class components. They are exclusively for functional components. Class components use `this.state` and lifecycle methods instead. `useState` inside a class component's `render` method violates the Rules of Hooks. If you need hook-based logic in a class component, the common workaround is to create a functional wrapper component or higher-order component that uses the hook and passes the values down as props.

</details>

---

### Q24

```jsx
function useLogger(value) {
  useEffect(() => {
    console.log('value changed to:', value);
  }, [value]);
}

function App() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');

  useLogger(count);
  useLogger(name);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={() => setName('Bob')}>Change Name</button>
    </>
  );
}
```

#### ❓ "Count" is clicked twice, then "Change Name" once. What is logged in total?

<details>
<summary>✅ Answer</summary>

```txt
// On mount:
value changed to: 0
value changed to: Alice

// After 1st count click:
value changed to: 1

// After 2nd count click:
value changed to: 2

// After name change:
value changed to: Bob
```

**Explanation:** Each `useLogger` call creates an independent `useEffect`. Changing `count` triggers only the first `useLogger`'s effect (the one tracking `count`). Changing `name` triggers only the second `useLogger`'s effect. They are completely independent — each tracks its own dep. This demonstrates that custom hooks can be called multiple times in a single component for reusable effect logic. 5 logs total.

</details>

---

### Q25

```jsx
function useEventLogger(eventName) {
  const countRef = useRef(0);

  useEffect(() => {
    const handler = () => {
      countRef.current += 1;
      console.log(`${eventName} fired ${countRef.current} times`);
    };
    window.addEventListener(eventName, handler);
    return () => window.removeEventListener(eventName, handler);
  }, [eventName]);
}

function App() {
  useEventLogger('click');
  useEventLogger('keydown');

  return <p>Click or type to see logs</p>;
}
```

#### ❓ The user clicks 3 times and presses 2 keys. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
click fired 1 times
click fired 2 times
click fired 3 times
keydown fired 1 times
keydown fired 2 times
```

**Explanation:** Each `useEventLogger` call creates an independent effect with its own `countRef`. The `click` logger has its own counter starting at 0. The `keydown` logger has its own separate counter starting at 0. They do not interfere. The `countRef` tracks how many times each specific event has fired, independently. Calling the same custom hook twice provides completely isolated instances — their internal state (refs, effects) are separate.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| State Isolation | Q1 – Q5 | independent state per call, same hook two calls same component, globalCount per instance, shared state misconception, parameter-configured hooks |
| Custom Hook Composition | Q6 – Q10 | composed hook semantics, two useCounter calls, debounce + fetch composition, localStorage + theme hook, usePrevious composition |
| useFetch Pattern | Q11 – Q15 | race condition with URL change, ignore flag fix, AbortController cleanup, single state object, parallel fetches |
| useLocalStorage | Q16 – Q20 | persistence across refresh, multiple instances don't sync, functional update stale closure, SSR-safe pattern, same key two instances |
| Edge Cases | Q21 – Q25 | conditional hook call error, hook outside component error, hook in class component error, multiple calls same component, independent ref per call |
