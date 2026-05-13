# React Custom Hooks

## Table of Contents

1. [What are Custom Hooks](#1-what-are-custom-hooks)
2. [Rules of Custom Hooks](#2-rules-of-custom-hooks)
3. [Why Custom Hooks](#3-why-custom-hooks)
4. [Common Custom Hook Patterns](#4-common-custom-hook-patterns)
5. [useFetch and useAsync](#5-usefetch-and-useasync)
6. [useLocalStorage](#6-uselocalstorage)
7. [useDebounce](#7-usedebounce)
8. [useWindowSize](#8-usewindowsize)
9. [usePrevious](#9-useprevious)
10. [useOnClickOutside](#10-useonclickoutside)
11. [useEventListener](#11-useeventlistener)
12. [useToggle](#12-usetoggle)
13. [Custom Hook Return Patterns](#13-custom-hook-return-patterns)
14. [State Isolation — Hooks Do Not Share State](#14-state-isolation--hooks-do-not-share-state)
15. [Testing Custom Hooks](#15-testing-custom-hooks)
16. [Custom Hook Composition](#16-custom-hook-composition)
17. [Common Mistakes](#17-common-mistakes)
18. [Best Practices](#18-best-practices)

---

## 1. What are Custom Hooks

A custom hook is a JavaScript function whose name **starts with `use`** and that can call other hooks. Custom hooks allow you to extract component logic into reusable functions.

### The Core Idea

```text
Before custom hooks — duplicated logic in every component:
  ComponentA:  const [data, setData] = useState(null);
               useEffect(() => { fetch(url).then(setData); }, [url]);

  ComponentB:  const [data, setData] = useState(null);
               useEffect(() => { fetch(url).then(setData); }, [url]);

After custom hooks — extracted into one place:
  function useFetch(url) {
    const [data, setData] = useState(null);
    useEffect(() => { fetch(url).then(setData); }, [url]);
    return data;
  }

  ComponentA: const data = useFetch(url);
  ComponentB: const data = useFetch(url);
```

### What Makes a Function a Custom Hook

1. The name starts with `use` (e.g., `useFetch`, `useToggle`, `useLocalStorage`)
2. It may call other hooks (built-in or custom)
3. It is a regular JavaScript function — no special React registration needed
4. React enforces hook rules for it because of the `use` prefix (detected by linters and React)

```jsx
// Custom hook — valid
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initialValue);
  return { count, increment, decrement, reset };
}

// Using the custom hook
function Counter() {
  const { count, increment, reset } = useCounter(10);
  return (
    <>
      <p>{count}</p>
      <button onClick={increment}>+</button>
      <button onClick={reset}>Reset</button>
    </>
  );
}
```

### When a Function is NOT a Custom Hook

A function that doesn't call any hooks is just a regular utility function — even if its name starts with `use`. But by convention, only functions that call hooks should have the `use` prefix.

---

## 2. Rules of Custom Hooks

Custom hooks obey the same Rules of Hooks as built-in hooks:

### Rule 1 — Only Call Hooks at the Top Level

Never call hooks inside loops, conditions, or nested functions:

❌ Wrong:

```jsx
function useBrokenHook(condition) {
  if (condition) {
    const [value, setValue] = useState(0); // conditional hook call!
  }
  return null;
}
```

✅ Correct:

```jsx
function useConditionalHook(condition) {
  const [value, setValue] = useState(0); // always called at top level
  // Use condition inside the hook body, not around the hook call
  const derived = condition ? value : null;
  return derived;
}
```

### Rule 2 — Only Call Hooks from React Functions

Custom hooks can only be called from:
- React functional components
- Other custom hooks

❌ Wrong — calling in a regular function:

```jsx
function regularFunction() {
  const [count, setCount] = useState(0); // Error!
}
```

✅ Correct:

```jsx
function useHookInHook() {
  const count = useCounter(); // calling a custom hook from another custom hook ✅
  return count;
}
```

### The `use` Prefix Rule

The `use` prefix is how React (and its linters) identifies custom hooks to enforce the Rules of Hooks. If you rename a hook to not start with `use`, the linter will no longer enforce hook rules inside it:

```jsx
// React / ESLint treats this as a custom hook — rules enforced
function useMyHook() { ... }

// React / ESLint treats this as a regular function — rules NOT enforced
function myHook() { ... }  // don't do this if it calls hooks
```

---

## 3. Why Custom Hooks

### Reusable Logic

The same logic can be shared across many components without any changes:

```jsx
// Used by 10 different components — fetch logic in one place
function useFetch(url) { ... }

function UserProfile({ userId }) {
  const { data, loading } = useFetch(`/api/users/${userId}`);
}

function ProductList({ categoryId }) {
  const { data, loading } = useFetch(`/api/products?cat=${categoryId}`);
}
```

### Separation of Concerns

Components focus on rendering. Custom hooks handle the logic:

```jsx
// Component only cares about rendering
function SearchBar() {
  const { query, setQuery, results, loading } = useSearch();

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {loading ? <Spinner /> : <ResultList results={results} />}
    </>
  );
}

// All state, effects, and logic in the hook
function useSearch() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!query) return;
    setLoading(true);
    fetchResults(query).then(r => {
      setResults(r);
      setLoading(false);
    });
  }, [query]);

  return { query, setQuery, results, loading };
}
```

### Testability

Custom hooks can be tested in isolation using `renderHook` from `@testing-library/react`:

```jsx
import { renderHook, act } from '@testing-library/react';

test('useCounter increments correctly', () => {
  const { result } = renderHook(() => useCounter(0));

  act(() => result.current.increment());

  expect(result.current.count).toBe(1);
});
```

### Consistency

Logic is defined once and behaves consistently everywhere. Bug fixes and improvements propagate automatically to all consumers.

---

## 4. Common Custom Hook Patterns

A summary of the patterns covered in depth in subsequent sections:

| Hook | Purpose |
|------|---------|
| `useFetch` / `useAsync` | Data fetching with loading/error state |
| `useLocalStorage` | State persisted to localStorage |
| `useDebounce` | Debounced value (delays fast-changing inputs) |
| `useWindowSize` | Current window dimensions |
| `usePrevious` | Previous value of a state/prop |
| `useOnClickOutside` | Detect clicks outside a specific element |
| `useEventListener` | Generic event listener with cleanup |
| `useToggle` | Boolean toggle with stable toggle function |

---

## 5. useFetch and useAsync

### Basic useFetch

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;

    let ignore = false;
    setLoading(true);
    setError(null);

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        if (!ignore) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!ignore) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => {
      ignore = true; // prevent race conditions on URL change
    };
  }, [url]);

  return { data, loading, error };
}
```

### Usage

```jsx
function UserCard({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <p>Error: {error}</p>;
  return <p>{user.name}</p>;
}
```

### useAsync — Generic Async Pattern

```jsx
function useAsync(asyncFn, immediate = true) {
  const [status, setStatus] = useState('idle');
  const [value, setValue] = useState(null);
  const [error, setError] = useState(null);

  const execute = useCallback(() => {
    setStatus('pending');
    setValue(null);
    setError(null);

    return asyncFn()
      .then(response => {
        setValue(response);
        setStatus('success');
      })
      .catch(error => {
        setError(error);
        setStatus('error');
      });
  }, [asyncFn]);

  useEffect(() => {
    if (immediate) execute();
  }, [execute, immediate]);

  return { execute, status, value, error };
}
```

---

## 6. useLocalStorage

### Implementation

```jsx
function useLocalStorage(key, initialValue) {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.warn('useLocalStorage read error:', error);
      return initialValue;
    }
  });

  const setValue = useCallback((value) => {
    try {
      // Allow value to be a function (like setState)
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.warn('useLocalStorage write error:', error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
}
```

### Usage

```jsx
function App() {
  const [name, setName] = useLocalStorage('userName', 'Anonymous');

  return (
    <input
      value={name}
      onChange={e => setName(e.target.value)}
      placeholder="Your name"
    />
  );
}
```

### Key Design Decisions

- Lazy initialization reads from localStorage only once (on first render)
- `try/catch` handles SSR environments (no `window`) and JSON parse errors
- `setValue` accepts a function (like `useState`'s setter) for consistency
- The key is included in the `useCallback` deps — if key changes, the setter updates correctly

---

## 7. useDebounce

### Implementation

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(handler); // cancel pending timeout when value changes
    };
  }, [value, delay]);

  return debouncedValue;
}
```

### How It Works

```text
User types: "r" → "re" → "rea" → "reac" → "react"
  Each keystroke cancels the previous timeout and starts a new one.
  Only after the user stops typing for `delay` ms does the debounced value update.

value changes     → effect cleanup cancels old timeout
                  → new timeout starts
delay passes      → setDebouncedValue(latestValue)
debouncedValue    → useEffect in consuming component runs only once
```

### Usage

```jsx
function SearchInput() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);

  useEffect(() => {
    if (debouncedQuery) {
      console.log('Searching for:', debouncedQuery);
      // fetchSearchResults(debouncedQuery);
    }
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

---

## 8. useWindowSize

### Implementation

```jsx
function useWindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    const handleResize = () => {
      setSize({
        width: window.innerWidth,
        height: window.innerHeight,
      });
    };

    window.addEventListener('resize', handleResize);

    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []); // runs once — listener is registered once and cleaned up on unmount

  return size;
}
```

### SSR-Safe Version

```jsx
function useWindowSize() {
  const [size, setSize] = useState(() => ({
    width: typeof window !== 'undefined' ? window.innerWidth : 0,
    height: typeof window !== 'undefined' ? window.innerHeight : 0,
  }));

  useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
}
```

### Usage

```jsx
function ResponsiveGrid() {
  const { width } = useWindowSize();
  const columns = width < 600 ? 1 : width < 900 ? 2 : 3;

  return (
    <div style={{ display: 'grid', gridTemplateColumns: `repeat(${columns}, 1fr)` }}>
      {items.map(item => <Card key={item.id} item={item} />)}
    </div>
  );
}
```

---

## 9. usePrevious

### Implementation

```jsx
function usePrevious(value) {
  const ref = useRef(undefined);

  useEffect(() => {
    ref.current = value;
  }); // no deps array — runs after every render

  return ref.current; // returns the value from the previous render
}
```

### How It Works — Render Timeline

```text
Render 1 (value = 0):
  ref.current = undefined (initial)
  usePrevious returns: undefined
  After render: effect runs → ref.current = 0

Render 2 (value = 1):
  ref.current = 0 (set by previous effect)
  usePrevious returns: 0    ← previous value ✅
  After render: effect runs → ref.current = 1
```

### Usage

```jsx
function UserCard({ userId }) {
  const prevUserId = usePrevious(userId);

  useEffect(() => {
    if (prevUserId !== undefined && prevUserId !== userId) {
      console.log(`User changed: ${prevUserId} → ${userId}`);
    }
  }, [userId, prevUserId]);

  return <div>User: {userId}</div>;
}
```

---

## 10. useOnClickOutside

### Implementation

```jsx
function useOnClickOutside(ref, handler) {
  useEffect(() => {
    const listener = (event) => {
      // Ignore if the click was inside the ref element
      if (!ref.current || ref.current.contains(event.target)) {
        return;
      }
      handler(event);
    };

    document.addEventListener('mousedown', listener);
    document.addEventListener('touchstart', listener);

    return () => {
      document.removeEventListener('mousedown', listener);
      document.removeEventListener('touchstart', listener);
    };
  }, [ref, handler]); // ref is stable; handler should be memoized
}
```

### Usage

```jsx
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useRef(null);

  useOnClickOutside(dropdownRef, () => setIsOpen(false));

  return (
    <div ref={dropdownRef}>
      <button onClick={() => setIsOpen(o => !o)}>Toggle</button>
      {isOpen && (
        <ul>
          <li>Option 1</li>
          <li>Option 2</li>
        </ul>
      )}
    </div>
  );
}
```

### Why the `contains` Check

```text
Without contains check:
  Clicking inside the dropdown fires the listener
  → handler runs → dropdown closes immediately after opening ❌

With contains check:
  ref.current.contains(event.target) returns true for clicks inside
  → listener returns early → dropdown stays open ✅
```

---

## 11. useEventListener

### Implementation

```jsx
function useEventListener(eventName, handler, element = window, options) {
  const savedHandler = useRef(handler);

  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const targetElement = element?.current ?? element;
    if (!targetElement?.addEventListener) return;

    const eventListener = (event) => savedHandler.current(event);
    targetElement.addEventListener(eventName, eventListener, options);

    return () => {
      targetElement.removeEventListener(eventName, eventListener, options);
    };
  }, [eventName, element, options]);
}
```

### Usage

```jsx
function App() {
  const [key, setKey] = useState('');

  useEventListener('keydown', (e) => {
    setKey(e.key);
  });

  return <p>Last key: {key}</p>;
}
```

### Why the Ref for Handler

```text
Without ref:
  handler is a new function reference on every render
  → useEffect with [handler] dep re-registers the listener on every render ← wasteful

With ref:
  The event listener always calls savedHandler.current
  savedHandler.current is updated on every render to the latest handler
  → Event listener registered only once ✅
  → Always calls the latest handler ✅
```

---

## 12. useToggle

### Implementation

```jsx
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => {
    setValue(v => !v);
  }, []); // stable — never recreated

  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return [value, toggle, { setTrue, setFalse }];
}
```

### Usage

```jsx
function ModalExample() {
  const [isOpen, toggleModal, { setFalse: closeModal }] = useToggle(false);

  return (
    <>
      <button onClick={toggleModal}>Toggle Modal</button>
      {isOpen && (
        <div className="modal">
          <p>Modal Content</p>
          <button onClick={closeModal}>Close</button>
        </div>
      )}
    </>
  );
}
```

### Why useCallback for toggle

The `toggle` function is wrapped in `useCallback` with `[]` deps — it is a stable reference that never changes. This allows passing `toggle` to `React.memo`-wrapped children or `useEffect` deps without triggering unnecessary re-renders or re-runs.

---

## 13. Custom Hook Return Patterns

### Pattern 1 — Array (like useState)

Best for hooks where position matters and you want to rename values at the call site:

```jsx
function useToggle(initial) {
  const [on, setOn] = useState(initial);
  return [on, () => setOn(v => !v)]; // like useState's [value, setter]
}

// Call site — easy to rename
const [isOpen, toggleOpen] = useToggle(false);
const [isActive, toggleActive] = useToggle(true);
```

### Pattern 2 — Object (named values)

Best for hooks returning many values where names clarify meaning:

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  // ...
  return { data, loading, error, refetch };
}

// Call site — names are explicit
const { data: user, loading, error } = useFetch('/api/user');
```

### Pattern 3 — Single Value

Best for simple value-only hooks:

```jsx
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    const handler = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  return width; // single value
}

// Call site
const width = useWindowWidth();
```

### Pattern 4 — Ref

Best for hooks that expose an imperative API (rare):

```jsx
function useScrollTop() {
  const ref = useRef(null);

  const scrollToTop = () => {
    ref.current?.scrollTo({ top: 0, behavior: 'smooth' });
  };

  return [ref, scrollToTop];
}

// Call site
const [containerRef, scrollToTop] = useScrollTop();
return <div ref={containerRef}>...</div>;
```

---

## 14. State Isolation — Hooks Do Not Share State

### The Critical Rule

Each call to a custom hook creates a **completely independent state**. Custom hooks are a mechanism for reusing logic, not sharing state.

```jsx
function useCounter() {
  const [count, setCount] = useState(0);
  return { count, increment: () => setCount(c => c + 1) };
}

function ComponentA() {
  const { count, increment } = useCounter(); // own state: count A
  return <button onClick={increment}>{count}</button>;
}

function ComponentB() {
  const { count, increment } = useCounter(); // own state: count B
  return <button onClick={increment}>{count}</button>;
}
```

`ComponentA` and `ComponentB` each have their own independent `count`. Clicking ComponentA's button does not affect ComponentB's counter.

```text
ComponentA → calls useCounter → gets its own useState(0) → count A = 0
ComponentB → calls useCounter → gets its own useState(0) → count B = 0

Click ComponentA's button:
  count A becomes 1
  count B remains 0 — completely independent ✅
```

### To Share State — Use Context

If two components need to share state, the state must live outside both components in a shared context or parent:

```jsx
const CounterContext = createContext(null);

function CounterProvider({ children }) {
  const counter = useCounter(); // ONE shared instance
  return <CounterContext.Provider value={counter}>{children}</CounterContext.Provider>;
}

function ComponentA() {
  const { count, increment } = useContext(CounterContext); // shared
}

function ComponentB() {
  const { count, increment } = useContext(CounterContext); // same state
}
```

---

## 15. Testing Custom Hooks

### Using renderHook

```jsx
import { renderHook, act } from '@testing-library/react';

function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  return {
    count,
    increment: () => setCount(c => c + 1),
    reset: () => setCount(initial),
  };
}

test('useCounter starts at initial value', () => {
  const { result } = renderHook(() => useCounter(5));
  expect(result.current.count).toBe(5);
});

test('useCounter increments correctly', () => {
  const { result } = renderHook(() => useCounter(0));

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});

test('useCounter resets to initial', () => {
  const { result } = renderHook(() => useCounter(10));

  act(() => {
    result.current.increment();
    result.current.reset();
  });

  expect(result.current.count).toBe(10);
});
```

### Testing with Context Dependencies

```jsx
test('useTheme reads from context', () => {
  const wrapper = ({ children }) => (
    <ThemeContext.Provider value="dark">
      {children}
    </ThemeContext.Provider>
  );

  const { result } = renderHook(() => useTheme(), { wrapper });
  expect(result.current).toBe('dark');
});
```

### Testing Async Hooks

```jsx
test('useFetch loads data', async () => {
  global.fetch = jest.fn().mockResolvedValue({
    ok: true,
    json: () => Promise.resolve({ name: 'Alice' }),
  });

  const { result, waitForNextUpdate } = renderHook(() => useFetch('/api/user'));

  expect(result.current.loading).toBe(true);

  await waitForNextUpdate();

  expect(result.current.loading).toBe(false);
  expect(result.current.data).toEqual({ name: 'Alice' });
});
```

---

## 16. Custom Hook Composition

Custom hooks can call other custom hooks — this is hook composition:

```jsx
// Low-level hooks
function useLocalStorage(key, defaultValue) { ... }
function useDebounce(value, delay) { ... }

// Composed hook — builds on two others
function useDebouncedLocalStorage(key, defaultValue, delay) {
  const [value, setValue] = useLocalStorage(key, defaultValue);
  const debouncedValue = useDebounce(value, delay);
  return [debouncedValue, setValue];
}

// High-level hook — composes two others
function useSearchWithHistory() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);
  const { data, loading } = useFetch(
    debouncedQuery ? `/api/search?q=${debouncedQuery}` : null
  );
  const [history, setHistory] = useLocalStorage('searchHistory', []);

  const search = (q) => {
    setQuery(q);
    setHistory(prev => [...new Set([q, ...prev])].slice(0, 10));
  };

  return { query, search, results: data, loading, history };
}
```

### Composition Benefits

```text
useFetch
  ↑
useSearchResults  ← composed from useFetch + useDebounce
  ↑
useProductSearch  ← composed from useSearchResults + useLocalStorage (history)
  ↑
ProductSearchPage  ← uses useProductSearch
```

Each layer adds a single concern. The result is testable and replaceable at each level.

---

## 17. Common Mistakes

### Mistake 1 — Forgetting Dependencies in useEffect Inside Custom Hook

❌ Wrong:

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, []); // url missing from deps — stale URL!
  return data;
}
```

✅ Correct:

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, [url]); // url included ✅
  return data;
}
```

### Mistake 2 — Returning Unstable References

❌ Wrong — new object on every call:

```jsx
function useUser(userId) {
  const [user, setUser] = useState(null);
  // ...
  return { user, userId, timestamp: Date.now() }; // new object every render
}
```

✅ Correct — stable references:

```jsx
function useUser(userId) {
  const [user, setUser] = useState(null);
  const refetch = useCallback(() => { /* ... */ }, [userId]);
  return { user, refetch }; // user is state, refetch is memoized
}
```

### Mistake 3 — Assuming Hooks Share State

❌ Wrong mental model:

```jsx
// Both components expect to share count — they don't!
function useSharedCounter() {
  const [count, setCount] = useState(0);
  return [count, () => setCount(c => c + 1)];
}

function A() { const [c] = useSharedCounter(); } // own count
function B() { const [c] = useSharedCounter(); } // own count — NOT shared
```

✅ Correct — use Context to share state.

### Mistake 4 — Calling Hooks Conditionally

❌ Wrong:

```jsx
function useConditionalHook(enabled) {
  if (!enabled) return null; // early return before all hooks!
  const [data, setData] = useState(null); // conditional hook!
  useEffect(() => { ... }, []);
  return data;
}
```

✅ Correct:

```jsx
function useConditionalHook(enabled) {
  const [data, setData] = useState(null);
  useEffect(() => {
    if (!enabled) return; // condition inside effect, not around hook call
    // fetch data...
  }, [enabled]);
  return enabled ? data : null;
}
```

---

## 18. Best Practices

### Naming Conventions

- Always start with `use`: `useForm`, `useAuth`, `useCart`
- Name after the capability it provides: `useWindowSize`, not `useResizeListener`
- For domain-specific hooks: `useUserProfile`, `useProductSearch`

### Return Value Decisions

| Scenario | Return Pattern |
|----------|----------------|
| Replacing useState (single value + setter) | Array `[value, setter]` |
| Multiple named values | Object `{ loading, data, error }` |
| Single derived value | Primitive or value directly |
| Exposing imperative methods | Object with methods |

### Design Principles

1. **Single responsibility** — each hook does one thing well. Compose hooks for complex behavior.

2. **Match the caller's expectations** — return values that work naturally with destructuring and React patterns.

3. **Always clean up** — if the hook registers event listeners, intervals, or subscriptions, return a cleanup function from `useEffect`.

4. **Handle errors gracefully** — provide an `error` state when the hook can fail.

5. **Accept configuration as parameters** — make hooks flexible with sensible defaults.

6. **Memoize callbacks** — use `useCallback` for functions returned from custom hooks that callers might pass to event handlers or useEffect deps.

7. **Document the hook** — at minimum, document the parameters and return values.

```jsx
/**
 * useDebounce — Returns a debounced version of the provided value.
 * @param {*} value - The value to debounce
 * @param {number} delay - Debounce delay in milliseconds
 * @returns {*} The debounced value (updates after `delay` ms of inactivity)
 */
function useDebounce(value, delay) { ... }
```
