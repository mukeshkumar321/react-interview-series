# React Hooks

## Table of Contents

- [1. What are Hooks?](#1-what-are-hooks)
- [2. Why Hooks Were Introduced](#2-why-hooks-were-introduced)
- [3. Rules of Hooks](#3-rules-of-hooks)
- [4. Hook Categories](#4-hook-categories)
- [5. useState Overview](#5-usestate-overview)
- [6. useEffect Overview](#6-useeffect-overview)
- [7. useRef Overview](#7-useref-overview)
- [8. useMemo Overview](#8-usememo-overview)
- [9. useCallback Overview](#9-usecallback-overview)
- [10. useContext Overview](#10-usecontext-overview)
- [11. useReducer Overview](#11-usereducer-overview)
- [12. Custom Hooks Overview](#12-custom-hooks-overview)
- [13. Hooks vs Class Lifecycle Methods](#13-hooks-vs-class-lifecycle-methods)
- [14. Mental Model for Hooks](#14-mental-model-for-hooks)
- [15. Hook Call Order](#15-hook-call-order)
- [16. Common Mistakes](#16-common-mistakes)
- [17. Best Practices](#17-best-practices)

---

## 1. What are Hooks?

Hooks are special functions introduced in React 16.8 that let you use React features — such as state, lifecycle behavior, context, and performance optimizations — directly inside functional components.

Before Hooks, only class components could manage state and perform side effects. Hooks changed this entirely by making functional components fully capable.

---

### Definition

A Hook is a JavaScript function that starts with the prefix `use` and connects a functional component to React's internal systems.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

---

### Key Characteristics

- Hooks are regular JavaScript functions with specific call rules
- They always start with the `use` prefix
- They can call other Hooks
- They cannot be used inside class components
- They do not replace the React component model — they enhance functional components

---

### Example Without and With Hooks

Without Hooks (class component):

```jsx
class Toggle extends React.Component {
  state = { isOn: false };

  render() {
    return (
      <button onClick={() => this.setState({ isOn: !this.state.isOn })}>
        {this.state.isOn ? "ON" : "OFF"}
      </button>
    );
  }
}
```

With Hooks (functional component):

```jsx
function Toggle() {
  const [isOn, setIsOn] = useState(false);

  return (
    <button onClick={() => setIsOn(!isOn)}>
      {isOn ? "ON" : "OFF"}
    </button>
  );
}
```

---

## 2. Why Hooks Were Introduced

Before Hooks, React developers faced three persistent problems with class components that Hooks were specifically designed to solve.

---

### Problem 1 — Reusing Stateful Logic Was Difficult

Sharing stateful behavior between components required patterns like Higher-Order Components (HOC) or Render Props. Both patterns added layers of nesting without modifying the component hierarchy in a readable way.

```jsx
// Wrapper hell with HOCs
<WithAuth>
  <WithTheme>
    <WithData>
      <MyComponent />
    </WithData>
  </WithTheme>
</WithAuth>
```

This made component trees hard to trace in React DevTools and caused "wrapper hell."

With Hooks, stateful logic is extracted into custom hooks and shared without wrapping:

```jsx
function MyComponent() {
  const { user } = useAuth();
  const { theme } = useTheme();
  const { data } = useData();
  // ...
}
```

---

### Problem 2 — Complex Components Were Hard to Understand

Class components grouped unrelated logic together inside the same lifecycle method, because that was the only way to run code at the right time.

```jsx
class Dashboard extends React.Component {
  componentDidMount() {
    fetchUser();           // concern A
    subscribeToEvents();   // concern B
    startAnalyticsTimer(); // concern C
  }

  componentWillUnmount() {
    unsubscribeFromEvents();
    clearAnalyticsTimer();
  }
}
```

With Hooks, each concern is isolated in its own `useEffect`:

```jsx
function Dashboard() {
  useEffect(() => { fetchUser(); }, []);

  useEffect(() => {
    subscribeToEvents();
    return () => unsubscribeFromEvents();
  }, []);

  useEffect(() => {
    startAnalyticsTimer();
    return () => clearAnalyticsTimer();
  }, []);
}
```

---

### Problem 3 — Classes Confused Both People and Machines

- The `this` keyword behaves differently from most other languages and requires careful binding
- Binding event handlers was verbose and error-prone
- Minifiers struggled to optimize class method names
- Hot reloading was unreliable with class components

---

### Side-by-Side Comparison

| Problem | Class Solution | Hooks Solution |
|---|---|---|
| Sharing stateful logic | HOC / Render Props | Custom Hooks |
| Grouping related code | Split across lifecycle methods | Multiple `useEffect` calls |
| `this` binding | `.bind(this)` or arrow class fields | Not needed |
| Code reuse between components | Wrapper components | Extract logic into hooks |
| Bundle optimization | Difficult to tree-shake | Functions are tree-shakeable |

---

## 3. Rules of Hooks

React enforces two fundamental rules for hooks. These rules are not arbitrary — they exist because of how React tracks hook state internally through a linked list.

---

### Rule 1 — Only Call Hooks at the Top Level

Do not call hooks inside conditions, loops, or nested functions. They must always be called in the same order on every render.

❌ Wrong

```jsx
function App({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // breaks hook order
  }
}
```

✅ Correct

```jsx
function App({ isLoggedIn }) {
  const [user, setUser] = useState(null); // always called

  if (isLoggedIn) {
    // use user here
  }
}
```

---

### Rule 2 — Only Call Hooks in React Functions

Hooks must be called from React functional components or custom hooks. Never from plain JavaScript functions, class components, or event handlers.

❌ Wrong

```js
function fetchData() {
  const [data, setData] = useState(null); // not a React function
}
```

✅ Correct

```jsx
function useFetchData() {
  const [data, setData] = useState(null);
  return data;
}
```

---

### Why These Rules Exist

React tracks hooks using an ordered linked list stored on each component's Fiber node. Every render must call hooks in exactly the same order so React can match each call to its stored state value.

```text
Render 1:
  useState  → slot 0  (count = 0)
  useEffect → slot 1  (lastDeps = [])
  useRef    → slot 2  (current = null)

Render 2:
  useState  → slot 0  ✅ matches
  useEffect → slot 1  ✅ matches
  useRef    → slot 2  ✅ matches
```

If a hook is inside a condition and the condition changes, all subsequent hooks read the wrong slot:

```text
Render 1 (condition = true):
  useState  → slot 0
  useEffect → slot 1  ← conditional hook
  useRef    → slot 2

Render 2 (condition = false):
  useState  → slot 0
  useRef    → slot 1  ← reads useEffect's slot by mistake ❌
```

React throws: `Rendered more hooks than during the previous render`.

---

### Enforcing Rules with ESLint

React provides an official ESLint plugin that detects rule violations at development time.

```bash
npm install eslint-plugin-react-hooks --save-dev
```

---

## 4. Hook Categories

React's built-in hooks are organized by what they connect a component to.

---

### State Hooks

Manage data that can change over time and trigger re-renders.

| Hook | Purpose |
|---|---|
| `useState` | Manage simple local state |
| `useReducer` | Manage complex state with actions and a reducer |

---

### Effect Hooks

Synchronize a component with external systems after rendering.

| Hook | Purpose |
|---|---|
| `useEffect` | Run after browser paint, for async side effects |
| `useLayoutEffect` | Run synchronously after DOM mutations, before paint |

---

### Ref Hooks

Hold a mutable value that persists across renders without triggering re-renders.

| Hook | Purpose |
|---|---|
| `useRef` | Store a mutable reference or access a DOM node |
| `useImperativeHandle` | Customize the ref value exposed to a parent component |

---

### Context Hooks

Read values from React Context without wrapping in a Consumer component.

| Hook | Purpose |
|---|---|
| `useContext` | Subscribe to a context value |

---

### Performance Hooks

Optimize rendering by memoizing values and function references.

| Hook | Purpose |
|---|---|
| `useMemo` | Memoize the result of an expensive computation |
| `useCallback` | Memoize a function reference |

---

### Additional Hooks

| Hook | Purpose |
|---|---|
| `useId` | Generate stable unique IDs for accessibility attributes |
| `useTransition` | Mark a state update as non-urgent (keeps UI responsive) |
| `useDeferredValue` | Defer re-rendering a non-urgent part of the UI |
| `useSyncExternalStore` | Subscribe to an external data store |
| `useDebugValue` | Display a custom label in React DevTools |

---

## 5. useState Overview

`useState` is the most fundamental hook. It adds local state to a functional component. When the state value changes, React re-renders the component.

---

### Syntax

```jsx
const [state, setState] = useState(initialValue);
```

- `state` — the current value for this render
- `setState` — function that schedules a state update and re-render
- `initialValue` — the value used only on the first render

---

### Example

```jsx
import { useState } from "react";

function Toggle() {
  const [isOn, setIsOn] = useState(false);

  return (
    <button onClick={() => setIsOn(!isOn)}>
      {isOn ? "ON" : "OFF"}
    </button>
  );
}
```

---

### Key Points

- `setState` does not update the variable immediately — it schedules a re-render
- The state value inside a render is a snapshot and never changes during that render
- Setting state to the same value (via `Object.is` comparison) skips re-render
- The initial value is ignored after the first render

---

## 6. useEffect Overview

`useEffect` lets you perform side effects in functional components. It runs after the component renders and the browser has painted the screen.

---

### Syntax

```jsx
useEffect(() => {
  // side effect

  return () => {
    // cleanup (optional)
  };
}, [dependencies]);
```

---

### Dependency Array Behavior

| Pattern | When the Effect Runs |
|---|---|
| No second argument | After every render |
| Empty array `[]` | Only after the initial mount |
| `[dep1, dep2]` | After mount, and whenever a listed dep changes |

---

### Example

```jsx
import { useEffect, useState } from "react";

function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setSeconds(prev => prev + 1);
    }, 1000);

    return () => clearInterval(id); // cleanup on unmount
  }, []);

  return <p>{seconds}s</p>;
}
```

---

### Cleanup Function

The cleanup function runs in two situations:

- Before the effect runs again (when dependencies change)
- When the component unmounts

Use it to cancel subscriptions, timers, fetch requests, or event listeners.

---

## 7. useRef Overview

`useRef` returns a mutable object whose `.current` property persists across renders. Changing `.current` does not trigger a re-render.

---

### Syntax

```jsx
const ref = useRef(initialValue);
```

---

### Use Case 1 — Mutable Value Without Re-render

```jsx
function Timer() {
  const timerIdRef = useRef(null);

  function start() {
    timerIdRef.current = setInterval(() => {
      console.log("tick");
    }, 1000);
  }

  function stop() {
    clearInterval(timerIdRef.current);
  }

  return (
    <>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </>
  );
}
```

---

### Use Case 2 — DOM Element Access

```jsx
function FocusInput() {
  const inputRef = useRef(null);

  function handleFocus() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />
      <button onClick={handleFocus}>Focus</button>
    </>
  );
}
```

---

### useRef vs useState

| Feature | useRef | useState |
|---|---|---|
| Causes re-render on change | No | Yes |
| Persists across renders | Yes | Yes |
| Mutable directly | Yes | No (use setter) |
| Typical use | DOM access, timers, previous values | UI data |

---

## 8. useMemo Overview

`useMemo` memoizes the result of an expensive computation. React returns the cached value on re-renders as long as the dependencies have not changed.

---

### Syntax

```jsx
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

---

### Example

```jsx
import { useMemo, useState } from "react";

function FilteredList({ items, filter }) {
  const filteredItems = useMemo(() => {
    console.log("filtering...");
    return items.filter(item => item.includes(filter));
  }, [items, filter]);

  return (
    <ul>
      {filteredItems.map(item => <li key={item}>{item}</li>)}
    </ul>
  );
}
```

---

### When to Use useMemo

- Computationally expensive operations (sorting, filtering large lists)
- Producing a stable object or array reference for child components or other hooks
- Avoiding redundant work during re-renders caused by unrelated state changes

---

### When NOT to Use useMemo

- Simple calculations — memoization overhead outweighs the benefit
- Values that change on every render anyway — caching provides no gain
- Premature optimization without profiling evidence

---

## 9. useCallback Overview

`useCallback` memoizes a function reference. It returns the same function instance between renders as long as the dependencies have not changed.

---

### Syntax

```jsx
const memoizedFn = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

---

### Why It Matters

Every time a component re-renders, all inline functions are recreated as new references. When passed to child components wrapped in `React.memo`, the new reference causes the child to re-render even though the logic is identical.

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // New reference every render — breaks React.memo on Child
  const handleClick = () => console.log("clicked");

  // Stable reference — React.memo on Child works correctly
  const stableHandleClick = useCallback(() => console.log("clicked"), []);

  return (
    <>
      <Child onClick={stableHandleClick} />
      <button onClick={() => setCount(count + 1)}>{count}</button>
    </>
  );
}

const Child = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>Click</button>;
});
```

---

### useMemo vs useCallback

| Hook | Returns | Primary Use |
|---|---|---|
| `useMemo` | Memoized computed value | Avoid expensive recalculations |
| `useCallback` | Memoized function reference | Stable props for memoized children |

`useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

---

## 10. useContext Overview

`useContext` subscribes a component to a React Context and returns its current value. It replaces the need for a Consumer wrapper component.

---

### Syntax

```jsx
const value = useContext(MyContext);
```

---

### Example

```jsx
import { createContext, useContext, useState } from "react";

const ThemeContext = createContext("light");

function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={theme}>
      <Toolbar />
      <button onClick={() => setTheme(t => t === "light" ? "dark" : "light")}>
        Toggle
      </button>
    </ThemeContext.Provider>
  );
}

function Toolbar() {
  return <ThemedButton />;
}

function ThemedButton() {
  const theme = useContext(ThemeContext);
  return <button className={theme}>Themed Button</button>;
}
```

---

### Context Re-render Behavior

Every component that calls `useContext(MyContext)` re-renders whenever the context value changes. To optimize performance, split contexts by update frequency or memoize the context value.

---

## 11. useReducer Overview

`useReducer` is an alternative to `useState` for managing state that involves multiple related values or complex update logic. It follows the reducer pattern: state transitions are described by dispatching action objects.

---

### Syntax

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

---

### Example

```jsx
import { useReducer } from "react";

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return { count: 0 };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </>
  );
}
```

---

### useState vs useReducer

| Scenario | Prefer |
|---|---|
| Simple boolean, string, or number | `useState` |
| Multiple related values updated together | `useReducer` |
| Next state depends on previous in complex ways | `useReducer` |
| Many distinct update actions | `useReducer` |
| Logic needs to be tested in isolation | `useReducer` |

---

## 12. Custom Hooks Overview

Custom hooks are JavaScript functions that start with `use` and can call other hooks inside. They let you extract reusable stateful logic out of components without changing component structure.

---

### Key Properties

- Each component using a custom hook gets its own isolated state
- Custom hooks share logic, not state instances
- They can accept arguments and return any value

---

### Example — useWindowWidth

```jsx
import { useState, useEffect } from "react";

function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() {
      setWidth(window.innerWidth);
    }

    window.addEventListener("resize", handleResize);

    return () => {
      window.removeEventListener("resize", handleResize);
    };
  }, []);

  return width;
}

function Component() {
  const width = useWindowWidth();
  return <p>Window width: {width}px</p>;
}
```

---

### Example — useFetch

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    fetch(url)
      .then(res => res.json())
      .then(json => {
        if (!cancelled) {
          setData(json);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err);
          setLoading(false);
        }
      });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}
```

---

## 13. Hooks vs Class Lifecycle Methods

This comparison is frequently tested in interviews. Hooks do not map one-to-one to lifecycle methods, but cover all the same behaviors.

---

### Comparison Table

| Class Lifecycle Method | Hook Equivalent |
|---|---|
| `constructor` | `useState(initialValue)` |
| `componentDidMount` | `useEffect(() => {}, [])` |
| `componentDidUpdate` | `useEffect(() => {}, [dep])` |
| `componentWillUnmount` | `useEffect(() => { return cleanup }, [])` |
| `shouldComponentUpdate` | `React.memo` + `useMemo` |
| `getDerivedStateFromProps` | Compute value during render |
| `getSnapshotBeforeUpdate` | `useLayoutEffect` |
| `componentDidCatch` | No direct hook — use Error Boundary class |

---

### componentDidMount Equivalent

```jsx
// Class
componentDidMount() {
  fetchData();
}

// Hook
useEffect(() => {
  fetchData();
}, []); // empty array → runs once after mount
```

---

### componentDidUpdate Equivalent

```jsx
// Class
componentDidUpdate(prevProps) {
  if (prevProps.id !== this.props.id) {
    fetchData(this.props.id);
  }
}

// Hook
useEffect(() => {
  fetchData(id);
}, [id]); // runs when id changes
```

---

### componentWillUnmount Equivalent

```jsx
// Class
componentWillUnmount() {
  subscription.unsubscribe();
}

// Hook
useEffect(() => {
  const sub = subscribe();
  return () => sub.unsubscribe(); // runs on unmount
}, []);
```

---

### Combining All Three

```jsx
useEffect(() => {
  const subscription = subscribeToData(id); // mount behavior

  return () => {
    subscription.cancel(); // unmount behavior, also runs before next effect
  };
}, [id]); // update behavior — re-runs when id changes
```

---

### useLayoutEffect vs useEffect

| Feature | useEffect | useLayoutEffect |
|---|---|---|
| Timing | After browser paint | After DOM update, before paint |
| Use case | Data fetching, subscriptions | DOM measurements, animations |
| Blocking | Non-blocking | Blocks paint |

---

## 14. Mental Model for Hooks

The wrong mental model is thinking of hooks as "lifecycle methods inside functions." The correct mental model is synchronization.

---

### Hooks as Synchronization

Each `useEffect` synchronizes your component with something external. React is the source of truth for UI; the effect keeps external systems aligned with that truth.

```text
React renders component
        ↓
Browser paints screen
        ↓
useEffect runs
        ↓
External system is in sync with React state
        ↓
State changes
        ↓
React re-renders
        ↓
useEffect cleanup runs (previous effect cancelled)
        ↓
useEffect runs again
        ↓
External system is re-synced
```

---

### State as a Snapshot

Each render captures a snapshot of all state values. The state variable inside a render always holds the value from the time that render started — even inside async callbacks that run later.

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);       // schedules re-render
    console.log(count);        // still 0 — this is the snapshot
  }
}
```

---

### Effects Describe Synchronization, Not Events

```jsx
// Wrong mental model: "run this code on mount"
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);

// Correct mental model:
// "Keep document.title synchronized with the count value"
```

---

## 15. Hook Call Order

React tracks every hook call using an index-based linked list per component instance stored on the Fiber node. The index increments by one for each hook call, in the order they appear in source code.

---

### Internal Storage Model

```text
Component Fiber Node
  ↓
Hook list:
  [0] useState  → { state: 0,    queue: [...] }
  [1] useEffect → { deps: [],    effect: fn  }
  [2] useRef    → { current: null            }
  [3] useMemo   → { value: 42,   deps: [5]   }
```

On every render, React walks this list in order. The nth hook call reads from the nth slot. This is why order must never change.

---

### Why Conditional Hooks Break

```jsx
function App({ isAdmin }) {
  const [name, setName] = useState(""); // always slot 0

  if (isAdmin) {
    // Sometimes slot 1, sometimes skipped
    // All hooks after this are shifted
    useEffect(() => { fetchAdminData(); }, []);
  }

  const ref = useRef(null); // slot 1 or slot 2 depending on condition
}
```

When `isAdmin` changes between renders, `useRef` reads from slot 1 but expects the data stored for slot 1 during the previous render — which was an effect, not a ref. React throws an error.

---

## 16. Common Mistakes

---

### Calling a Hook Conditionally

❌ Wrong

```jsx
function App({ show }) {
  if (show) {
    const [value, setValue] = useState("");
  }
}
```

---

### Calling a Hook in a Loop

❌ Wrong

```jsx
function App() {
  const items = [1, 2, 3];
  items.forEach(item => {
    const [val, setVal] = useState(item); // error
  });
}
```

---

### Calling a Hook Inside an Event Handler

❌ Wrong

```jsx
function App() {
  function handleClick() {
    const [count, setCount] = useState(0); // error
  }
  return <button onClick={handleClick}>Click</button>;
}
```

---

### Missing Dependency in useEffect

❌ Wrong

```jsx
useEffect(() => {
  fetchData(userId); // userId is used but not listed
}, []);
```

✅ Correct

```jsx
useEffect(() => {
  fetchData(userId);
}, [userId]);
```

---

### Stale Closure in useEffect

❌ Wrong

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // count is stale — always 0
  }, 1000);
  return () => clearInterval(id);
}, []); // count not in deps
```

✅ Correct

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // functional update — always fresh
  }, 1000);
  return () => clearInterval(id);
}, []);
```

---

### Returning a Non-Function from useEffect

❌ Wrong

```jsx
useEffect(async () => {
  const data = await fetchData(); // async function returns Promise, not cleanup fn
});
```

✅ Correct

```jsx
useEffect(() => {
  async function load() {
    const data = await fetchData();
  }
  load();
}, []);
```

---

## 17. Best Practices

---

### Extract Stateful Logic into Custom Hooks

Keeping component bodies clean by moving complex stateful logic into named custom hooks.

```jsx
// Before — mixed concerns in component
function UserPage({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchUser(userId).then(data => {
      setUser(data);
      setLoading(false);
    });
  }, [userId]);

  if (loading) return <p>Loading...</p>;
  return <p>{user.name}</p>;
}

// After — concern extracted
function UserPage({ userId }) {
  const { user, loading } = useUser(userId);

  if (loading) return <p>Loading...</p>;
  return <p>{user.name}</p>;
}
```

---

### Separate Concerns with Multiple Effects

One effect per logical concern makes code easier to read and maintain.

```jsx
// Good
useEffect(() => {
  subscribeToEvents();
  return () => unsubscribeFromEvents();
}, []);

useEffect(() => {
  document.title = title;
}, [title]);

useEffect(() => {
  analytics.track("page_view", { userId });
}, [userId]);
```

---

### Use Functional Updates When New State Depends on Previous State

```jsx
// Correct
setCount(prev => prev + 1);

// Risky — stale if batched or used in closures
setCount(count + 1);
```

---

### Use Lazy Initialization for Expensive Initial State

```jsx
// Wrong — runs expensiveCompute on every render
const [data, setData] = useState(expensiveCompute());

// Correct — runs expensiveCompute only on first render
const [data, setData] = useState(() => expensiveCompute());
```

---

### Use useRef for Values That Should Not Trigger Re-render

Timer IDs, interval IDs, previous render values, and DOM node references should use `useRef`.

---

### Keep Dependency Arrays Honest

Always list every value that an effect reads from the component scope. Use the `react-hooks/exhaustive-deps` ESLint rule to catch missing dependencies automatically.

---

### Do Not Over-Memoize

`useMemo` and `useCallback` add complexity and memory cost. Only apply them when:

- You have measured a real performance problem
- You need referential stability for `React.memo` or another hook's dependency array
- The computation is genuinely expensive
