# React useEffect Hook

## Table of Contents

1. [What is useEffect](#1-what-is-useeffect)
2. [Syntax](#2-syntax)
3. [When useEffect Runs](#3-when-useeffect-runs)
4. [Dependency Array Deep Dive](#4-dependency-array-deep-dive)
5. [Cleanup Function](#5-cleanup-function)
6. [useEffect vs useLayoutEffect](#6-useeffect-vs-uselayouteffect)
7. [Stale Closures in useEffect](#7-stale-closures-in-useeffect)
8. [Race Conditions](#8-race-conditions)
9. [Async useEffect](#9-async-useeffect)
10. [Data Fetching Pattern](#10-data-fetching-pattern)
11. [Event Listener Pattern](#11-event-listener-pattern)
12. [Interval Pattern](#12-interval-pattern)
13. [Infinite Loop Causes](#13-infinite-loop-causes)
14. [useEffect and Strict Mode](#14-useeffect-and-strict-mode)
15. [Multiple useEffect Hooks](#15-multiple-useeffect-hooks)
16. [Lifecycle Method Equivalents](#16-lifecycle-method-equivalents)
17. [Common Mistakes](#17-common-mistakes)
18. [Best Practices](#18-best-practices)

---

## 1. What is useEffect

`useEffect` is a React Hook that lets you synchronize a component with an external system. The word "external" is key — useEffect is the escape hatch from the pure rendering model into the world of side effects.

### What counts as a side effect

- Fetching data from a network
- Setting up a subscription (WebSocket, event emitter)
- Manually reading from or writing to the DOM
- Starting a timer (`setTimeout`, `setInterval`)
- Logging to an analytics service
- Integrating with a non-React third-party library

### What useEffect is NOT for

useEffect is not for computing derived state or transforming data — that belongs in the render itself. It is also not the right tool for synchronous DOM measurements that must happen before the browser paints; use `useLayoutEffect` for that.

### The mental model

Before Hooks, side effects were scattered across lifecycle methods: `componentDidMount`, `componentDidUpdate`, and `componentWillUnmount`. useEffect unifies all three into a single API driven by synchronization rather than lifecycle events.

Instead of asking "when do I run this code?", ask "with what external system should this component stay synchronized, and what values does that synchronization depend on?"

```jsx
import { useEffect } from 'react';

function ChatRoom({ roomId }) {
  useEffect(() => {
    // Connect to the room identified by roomId
    const connection = createConnection(roomId);
    connection.connect();

    // Disconnect when roomId changes or component unmounts
    return () => {
      connection.disconnect();
    };
  }, [roomId]); // Re-run whenever roomId changes
}
```

---

## 2. Syntax

### Basic signature

```js
useEffect(setup, dependencies?)
```

- `setup` — A function containing the side-effect logic. It may optionally return a cleanup function.
- `dependencies` — An optional array of reactive values that the effect depends on.

### The three forms

#### Form 1: No dependency array

```jsx
useEffect(() => {
  // Runs after EVERY render
  document.title = 'Updated';
});
```

Runs after the initial render and after every subsequent re-render. Use this when the effect must reflect every state and prop change, but be aware it runs very frequently.

#### Form 2: Empty dependency array

```jsx
useEffect(() => {
  // Runs only after the initial mount
  analytics.trackPageView();
}, []);
```

Runs once, after the component first appears in the DOM. Equivalent to `componentDidMount`. The cleanup (if any) runs on unmount.

#### Form 3: Array with specific dependencies

```jsx
useEffect(() => {
  // Runs on mount AND whenever userId changes
  fetchUserProfile(userId);
}, [userId]);
```

Runs after the initial render and again whenever any value in the dependency array changes between renders. React uses `Object.is` to compare each dependency against its previous value.

### Return value of useEffect

`useEffect` itself always returns `undefined`. The cleanup function is returned from the `setup` callback, not from `useEffect` itself.

```jsx
useEffect(() => {
  const id = setInterval(tick, 1000);

  return () => clearInterval(id); // returned from setup, not from useEffect
}, []);
```

---

## 3. When useEffect Runs

### Execution timing

`useEffect` runs asynchronously after the browser has painted the screen. This is deliberate: the effect does not block the browser from updating the display, so the user always sees a responsive UI.

```text
Component renders
       ↓
React commits changes to the DOM
       ↓
Browser paints the screen  ← user sees the update here
       ↓
useEffect callback fires   ← effect runs here
```

### Contrast with useLayoutEffect

`useLayoutEffect` fires synchronously after React updates the DOM but before the browser paints.

```text
Component renders
       ↓
React commits changes to the DOM
       ↓
useLayoutEffect fires      ← runs here, synchronously
       ↓
Browser paints the screen
       ↓
useEffect fires            ← runs here, asynchronously
```

Because `useLayoutEffect` is synchronous and blocks painting, it can cause visual glitches if misused. Reserve it for cases where you need to measure or mutate the DOM before the user sees it (e.g., tooltip positioning, scroll restoration).

### The sequence in practice

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  console.log('1. render');

  useEffect(() => {
    console.log('3. effect fires');
  });

  return (
    <button onClick={() => setCount(c => c + 1)}>
      {/* 2. browser paints */}
      {count}
    </button>
  );
}

// Console output on first render:
// 1. render
// (browser paints the button)
// 3. effect fires
```

---

## 4. Dependency Array Deep Dive

### Object.is comparison

React compares each dependency using `Object.is`, which is similar to `===` with two differences:

| Expression | `===` result | `Object.is` result |
|---|---|---|
| `NaN` vs `NaN` | `false` | `true` |
| `+0` vs `-0` | `true` | `false` |

For practical purposes:
- Primitive values (numbers, strings, booleans) are compared by value.
- Objects, arrays, and functions are compared by reference.

### No dependency array — runs on every render

```jsx
useEffect(() => {
  console.log('runs after every render');
});
```

Legitimate when the effect must track every render, but often a bug caused by forgetting to add the array.

### Empty array — runs on mount only

```jsx
useEffect(() => {
  console.log('runs once after mount');
}, []);
```

The effect sees only the values that existed at mount time. Reading state or props inside without listing them in the dependency array creates a stale closure (see Section 7).

### Specific dependencies — runs on mount and on dep change

```jsx
useEffect(() => {
  document.title = `Count: ${count}`;
}, [count]);
```

The effect fires after the initial render, then again whenever `count` changes. If multiple deps are listed, the effect fires when any one of them changes between renders.

### The exhaustive-deps ESLint rule

The `eslint-plugin-react-hooks` package provides `react-hooks/exhaustive-deps`. It statically analyzes your effect and warns when a reactive value is used inside the effect but not listed in the dependency array.

```bash
npm install --save-dev eslint-plugin-react-hooks
```

```js
// .eslintrc.js
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

### Objects and functions as dependencies

A plain object or function literal created inside the component body is a new reference on every render. Listing it in the dependency array causes the effect to re-run on every render.

```jsx
// ❌ Wrong — options is a new object reference on every render
function SearchResults({ query }) {
  const options = { method: 'GET', query };

  useEffect(() => {
    fetchResults(options);
  }, [options]); // runs on every render
}
```

```jsx
// ✅ Correct — list the primitive values the effect actually needs
function SearchResults({ query }) {
  useEffect(() => {
    const options = { method: 'GET', query };
    fetchResults(options);
  }, [query]); // runs only when query changes
}
```

---

## 5. Cleanup Function

### Purpose

The cleanup function returned from a useEffect callback serves two purposes:

1. Run before the next execution of the same effect (when dependencies change between renders).
2. Run when the component unmounts.

This lets you undo whatever the effect set up — disconnect a socket, cancel a request, remove an event listener, clear a timer.

### Syntax

```jsx
useEffect(() => {
  // setup
  const subscription = subscribe(topic);

  return () => {
    // cleanup
    subscription.unsubscribe();
  };
}, [topic]);
```

### When cleanup runs — timeline

The cleanup does NOT run after the current effect completes. It runs before the next invocation of the same effect and on unmount.

```text
Mount
  ↓
Effect A runs (topic = "sports")
  ↓
topic prop changes to "finance"
  ↓
Cleanup A runs   ← reverses "sports" subscription
  ↓
Effect B runs (topic = "finance")
  ↓
Component unmounts
  ↓
Cleanup B runs   ← reverses "finance" subscription
```

### clearInterval example

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setTime(Date.now());
  }, 1000);

  return () => clearInterval(id); // prevents interval stacking
}, []);
```

### removeEventListener example

```jsx
useEffect(() => {
  function handleResize() {
    setWidth(window.innerWidth);
  }

  window.addEventListener('resize', handleResize);

  return () => window.removeEventListener('resize', handleResize);
}, []);
```

### AbortController example

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch('/api/data', { signal: controller.signal })
    .then(res => res.json())
    .then(data => setData(data))
    .catch(err => {
      if (err.name !== 'AbortError') setError(err);
    });

  return () => controller.abort();
}, []);
```

### Consequences of missing cleanup

| Resource | Without cleanup | With cleanup |
|---|---|---|
| `setInterval` | Multiple intervals stack up each re-mount | One interval at a time |
| `addEventListener` | Duplicate handlers fire on every event | One handler registered |
| WebSocket | Connection stays open after unmount | Connection closed on unmount |
| Async fetch | `setState` called on unmounted component | Request cancelled |

---

## 6. useEffect vs useLayoutEffect

### Timing comparison

| Hook | When it fires | Blocks paint? | Thread |
|---|---|---|---|
| `useEffect` | After browser paint | No | Asynchronous |
| `useLayoutEffect` | After DOM update, before browser paint | Yes | Synchronous |

### Use useEffect when

- Fetching data from an API
- Setting up a subscription or WebSocket
- Logging analytics events
- Starting a timer
- Any side effect that does not need to read or write the DOM synchronously before the user sees it

### Use useLayoutEffect when

- Measuring a DOM element's size or position before the browser paints
- Repositioning a tooltip or popover based on element dimensions
- Preventing a visible flash caused by a synchronous DOM mutation
- Integrating with a library that expects synchronous DOM reads right after render

### Visual flow comparison

```text
useEffect:
  render → DOM updated → [browser paints] → effect fires

useLayoutEffect:
  render → DOM updated → [effect fires] → browser paints
```

### Code comparison

```jsx
// useEffect — tooltip position flickers because paint happens first
useEffect(() => {
  const rect = tooltipRef.current.getBoundingClientRect();
  setPosition({ top: rect.bottom, left: rect.left });
}, []);

// useLayoutEffect — position is set before user sees anything
useLayoutEffect(() => {
  const rect = tooltipRef.current.getBoundingClientRect();
  setPosition({ top: rect.bottom, left: rect.left });
}, []);
```

### Performance warning

`useLayoutEffect` runs synchronously and blocks the browser from painting. Heavy computation inside it will delay the paint and degrade perceived performance. Default to `useEffect` and only reach for `useLayoutEffect` when you have a concrete visible flicker to fix.

### Server-Side Rendering note

`useLayoutEffect` does not run on the server (in Next.js / SSR). React will warn: "useLayoutEffect does nothing on the server." For SSR-compatible code, use `useEffect` or guard with `if (typeof window !== 'undefined')`.

---

## 7. Stale Closures in useEffect

### What is a stale closure

A closure captures variables from its surrounding scope at the time it is created. When a useEffect callback is created during a render, it captures the values of all variables as they were at that render. If the dependency array prevents the effect from re-running on subsequent renders, the effect continues to hold references to the original (stale) values even as state updates.

### The classic stale closure problem

```jsx
// ❌ Wrong — stale closure
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // count is always 0 — captured at mount time
    }, 1000);

    return () => clearInterval(id);
  }, []); // empty deps — effect never re-runs

  return <p>{count}</p>;
}
// count will always show 1, never increments beyond that
```

### Why this happens

At mount time, the effect closure captures `count = 0`. The interval callback is a new function created inside that closure. Every time it fires, it computes `0 + 1 = 1` and sets count to `1`. Count is already `1` so React bails out. The counter is stuck.

### Fix 1: Add the value to the dependency array

Adding `count` to the dependency array causes the effect to re-run (and re-create the interval) whenever `count` changes. The closure now always captures the latest `count`.

```jsx
// ✅ Correct — but re-creates the interval on every count change
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1);
  }, 1000);

  return () => clearInterval(id);
}, [count]);
```

This is correct but suboptimal for intervals: the interval is torn down and re-created every second.

### Fix 2: Functional update form of setState

The functional update form `setState(prev => prev + 1)` receives the latest state value as an argument. The closure does not need to capture `count` at all, so the dependency array can remain empty.

```jsx
// ✅ Correct — effect runs once, always increments from latest value
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // prev is always the current value
  }, 1000);

  return () => clearInterval(id);
}, []);
```

### Fix 3: useRef for always-fresh values

Store the latest value in a ref. Refs are mutable objects that persist across renders and do not cause re-renders when mutated. The effect can always read the latest value through `ref.current` without listing it as a dependency.

```jsx
// ✅ Correct — ref always holds the latest callback
function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  // Keep the ref updated after every render
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  // Set up the interval; never re-creates it due to callback changes
  useEffect(() => {
    if (delay === null) return;
    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}
```

### Stale closure in event handlers inside effects

```jsx
// ❌ Wrong — handler captures stale value
useEffect(() => {
  function handleClick() {
    console.log(count); // always logs the value of count at effect creation time
  }

  window.addEventListener('click', handleClick);
  return () => window.removeEventListener('click', handleClick);
}, []); // count is missing from deps
```

```jsx
// ✅ Correct — add count to deps so handler is recreated when count changes
useEffect(() => {
  function handleClick() {
    console.log(count);
  }

  window.addEventListener('click', handleClick);
  return () => window.removeEventListener('click', handleClick);
}, [count]);
```

---

## 8. Race Conditions

### The problem

When a component triggers multiple async requests (e.g., as the user types in a search field), responses can arrive out of order. If an earlier request's response arrives last, it overwrites the correct data.

```jsx
// ❌ Race condition — last response wins, not last request
function SearchResults({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => setResults(data)); // might display stale data
  }, [query]);
}
```

```text
query changes: "re" → "rea" → "reac"

Request 1 (query="re")   → sent ─────────────────────→ slow, arrives LAST  ← sets wrong data ❌
Request 2 (query="rea")  → sent ─────────→ arrives second                   ← sets correct data
Request 3 (query="reac") → sent ──→ arrives FIRST
                                    ↑
                   UI briefly shows correct, then reverts when Request 1 arrives ❌
```

### Fix 1: Boolean ignore flag

A boolean `ignore` is set to `true` in the cleanup. The response handler checks the flag before updating state.

```jsx
// ✅ Correct — ignore stale responses
useEffect(() => {
  let ignore = false;

  fetch(`/api/search?q=${query}`)
    .then(res => res.json())
    .then(data => {
      if (!ignore) setResults(data);
    });

  return () => {
    ignore = true; // discard the in-flight response when query changes
  };
}, [query]);
```

### Fix 2: AbortController

`AbortController` cancels the network request itself, freeing browser resources. It is the preferred approach when using `fetch`.

```jsx
// ✅ Correct — cancel the request on cleanup
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => setResults(data))
    .catch(err => {
      if (err.name === 'AbortError') return; // expected on cleanup, ignore
      setError(err);
    });

  return () => controller.abort();
}, [query]);
```

### Comparison

| Approach | Cancels network request | Works with non-fetch async | Browser support |
|---|---|---|---|
| Ignore flag | No (request continues in background) | Yes | All |
| AbortController | Yes | No (fetch only) | Modern browsers |

---

## 9. Async useEffect

### Why you cannot make the callback async directly

`useEffect` expects its `setup` callback to either return nothing or return a synchronous cleanup function. An `async` function always returns a Promise. React cannot use a Promise as a cleanup function — it is silently discarded — and any cleanup you try to return from inside the async function never runs.

```jsx
// ❌ Wrong — async callback returns a Promise, cleanup is lost
useEffect(async () => {
  const data = await fetchData();
  setData(data);
  return () => cleanup(); // ❌ this runs inside the Promise resolution, not as a cleanup
}, []);
```

### Pattern 1: Named async function inside, called immediately

```jsx
// ✅ Correct
useEffect(() => {
  async function load() {
    const data = await fetchData();
    setData(data);
  }

  load(); // returns a Promise but useEffect ignores it

  // cleanup can still be returned normally
}, []);
```

The outer callback is synchronous and returns `undefined`. The cleanup mechanism works correctly.

### Pattern 2: With cleanup

```jsx
// ✅ Correct — async with request cancellation
useEffect(() => {
  const controller = new AbortController();

  async function load() {
    try {
      const res  = await fetch('/api/data', { signal: controller.signal });
      const data = await res.json();
      setData(data);
    } catch (err) {
      if (err.name !== 'AbortError') setError(err);
    }
  }

  load();

  return () => controller.abort(); // outer function returns cleanup
}, []);
```

### Pattern 3: IIFE

```jsx
// ✅ Correct — immediately invoked async expression
useEffect(() => {
  (async () => {
    const data = await fetchData();
    setData(data);
  })();
}, []);
```

Equivalent to Pattern 1. Pattern 1 is generally preferred for readability and ease of testing.

---

## 10. Data Fetching Pattern

### Complete pattern with loading, error, and data states

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user,    setUser]    = useState(null);
  const [loading, setLoading] = useState(true);
  const [error,   setError]   = useState(null);

  useEffect(() => {
    let ignore = false;
    const controller = new AbortController();

    setLoading(true);
    setError(null);

    async function fetchUser() {
      try {
        const res = await fetch(`/api/users/${userId}`, {
          signal: controller.signal,
        });

        if (!res.ok) throw new Error(`HTTP ${res.status}`);

        const data = await res.json();

        if (!ignore) {
          setUser(data);
          setLoading(false);
        }
      } catch (err) {
        if (err.name === 'AbortError') return;
        if (!ignore) {
          setError(err.message);
          setLoading(false);
        }
      }
    }

    fetchUser();

    return () => {
      ignore = true;
      controller.abort();
    };
  }, [userId]);

  if (loading) return <p>Loading...</p>;
  if (error)   return <p>Error: {error}</p>;
  if (!user)   return null;

  return <h1>{user.name}</h1>;
}
```

### Why both `ignore` and `AbortController`

`AbortController` cancels the network request. The `ignore` flag handles edge cases where the response is already cached in memory: the abort signal might arrive after the fetch resolves but before the `.then` handler checks for it. Using both is belt-and-suspenders.

### Extracting into a custom hook

```jsx
function useFetch(url) {
  const [data,    setData]    = useState(null);
  const [loading, setLoading] = useState(true);
  const [error,   setError]   = useState(null);

  useEffect(() => {
    if (!url) return;

    let ignore = false;
    const controller = new AbortController();

    setLoading(true);

    async function load() {
      try {
        const res  = await fetch(url, { signal: controller.signal });
        const json = await res.json();
        if (!ignore) { setData(json); setLoading(false); }
      } catch (err) {
        if (err.name === 'AbortError') return;
        if (!ignore) { setError(err); setLoading(false); }
      }
    }

    load();

    return () => { ignore = true; controller.abort(); };
  }, [url]);

  return { data, loading, error };
}
```

---

## 11. Event Listener Pattern

### The correct pattern

```jsx
useEffect(() => {
  function handleKeyDown(event) {
    if (event.key === 'Escape') closeModal();
  }

  document.addEventListener('keydown', handleKeyDown);

  return () => document.removeEventListener('keydown', handleKeyDown);
}, [closeModal]); // list closeModal if it changes between renders
```

### Why cleanup is critical

Without removing the listener on unmount, the handler persists after the component is gone. The handler holds a reference to `closeModal`, which holds references to the component's state and props. This is a memory leak and also causes errors when the handler fires and tries to update state on an unmounted component.

### Window resize example

```jsx
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    function handleResize() {
      setWidth(window.innerWidth);
    }

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return width;
}
```

### Online / offline status example

```jsx
function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useEffect(() => {
    const setTrue  = () => setOnline(true);
    const setFalse = () => setOnline(false);

    window.addEventListener('online',  setTrue);
    window.addEventListener('offline', setFalse);

    return () => {
      window.removeEventListener('online',  setTrue);
      window.removeEventListener('offline', setFalse);
    };
  }, []);

  return online;
}
```

---

## 12. Interval Pattern

### The basic pattern

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setTick(t => t + 1); // functional update — no stale closure
  }, 1000);

  return () => clearInterval(id);
}, []);
```

### Why the functional update matters

`setTick(t => t + 1)` does not require `tick` to be in the dependency array. If you wrote `setTick(tick + 1)`, you would need `tick` as a dependency, which would clear and re-start the interval on every tick — technically correct but the interval restarts every second instead of firing continuously, and the timing can drift.

### The useInterval custom hook (Dan Abramov pattern)

This pattern separates the interval scheduling concern from the callback logic. It lets you change the callback without restarting the interval.

```jsx
function useInterval(callback, delay) {
  const savedCallback = useRef(callback);

  // Keep the ref updated with the latest callback after every render
  useEffect(() => {
    savedCallback.current = callback;
  }, [callback]);

  // Set up the interval — only restarts when delay changes
  useEffect(() => {
    if (delay === null) return; // null pauses the interval

    const id = setInterval(() => savedCallback.current(), delay);
    return () => clearInterval(id);
  }, [delay]);
}

// Usage
function Timer() {
  const [count, setCount] = useState(0);
  const [paused, setPaused] = useState(false);

  useInterval(() => {
    setCount(c => c + 1);
  }, paused ? null : 1000);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setPaused(p => !p)}>
        {paused ? 'Resume' : 'Pause'}
      </button>
    </div>
  );
}
```

---

## 13. Infinite Loop Causes

### Cause 1: Object literal in dependency array

```jsx
// ❌ Infinite loop — new object reference created on every render
function Profile({ userId }) {
  const config = { userId, expand: true }; // new reference every render

  useEffect(() => {
    fetchProfile(config);
  }, [config]); // config !== config (prev) → effect fires → setState → re-render → new config → ...
}
```

```jsx
// ✅ Fix — list the primitive values the effect actually needs
function Profile({ userId }) {
  useEffect(() => {
    fetchProfile({ userId, expand: true }); // object created inside effect, not in deps
  }, [userId]);
}
```

### Cause 2: Inline function in dependency array

```jsx
// ❌ Infinite loop — new function reference on every render
function Component({ onLoad }) {
  useEffect(() => {
    onLoad();
  }, [onLoad]); // if onLoad is defined inline in the parent, new ref each render
}
```

```jsx
// ✅ Fix — memoize the function at the call site with useCallback
function Parent() {
  const handleLoad = useCallback(() => {
    console.log('loaded');
  }, []); // stable reference

  return <Component onLoad={handleLoad} />;
}
```

### Cause 3: setState inside effect without proper guard

```jsx
// ❌ Infinite loop — unconditional setState triggers re-render triggers effect
useEffect(() => {
  setCount(count + 1); // no condition, no limit
});
```

```jsx
// ✅ Fix — add a condition or restrict with dependency array
useEffect(() => {
  if (count < 10) setCount(count + 1);
}, [count]);
```

### Cause 4: Deriving state inside an effect that depends on that state

```jsx
// ❌ Infinite loop
const [fullName, setFullName] = useState('');

useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName, fullName]); // fullName changes → effect fires → fullName changes → ...
```

```jsx
// ✅ Fix — compute derived state during render, not in an effect
const fullName = `${firstName} ${lastName}`;
```

---

## 14. useEffect and Strict Mode

### React 18 Strict Mode behavior in development

In development, React 18 intentionally mounts every component twice:

1. Component mounts → effect fires
2. Component unmounts → cleanup fires
3. Component mounts again → effect fires again

This double-mount verifies that your cleanup function correctly reverses the effect. If your app misbehaves after the second mount, the cleanup is incomplete.

```text
Development (StrictMode):
  Mount  → Effect runs
  Unmount → Cleanup runs
  Mount  → Effect runs again  ← React is testing cleanup correctness
  Unmount → Cleanup runs (when navigating away)

Production:
  Mount  → Effect runs
  Unmount → Cleanup runs
```

### What double-mounting catches

- Subscriptions not cleaned up (they stack after the second mount, causing duplicate messages)
- Non-idempotent effects (effects that break when run twice, e.g., writing to a global)
- Missing cleanup for timers, event listeners, WebSocket connections

### Writing effects that survive double mounting

```jsx
// ❌ Wrong — connection created twice, only disconnected once on unmount
useEffect(() => {
  const socket = new WebSocket('wss://example.com');
  socket.onmessage = handleMessage;
  // no cleanup
}, []);

// ✅ Correct — each mount creates a connection; each unmount closes it
useEffect(() => {
  const socket = new WebSocket('wss://example.com');
  socket.onmessage = handleMessage;

  return () => socket.close();
}, []);
```

### Production behavior

In production, components mount exactly once. Strict Mode double-mounting is a development-only mechanism.

---

## 15. Multiple useEffect Hooks

### Each effect handles one concern

React allows multiple `useEffect` calls in a single component. This is the preferred style when a component has multiple independent side effects.

```jsx
function UserDashboard({ userId, theme }) {
  // Concern 1: fetch user data when userId changes
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);

  // Concern 2: apply theme to document
  useEffect(() => {
    document.body.className = theme;
    return () => { document.body.className = ''; };
  }, [theme]);

  // Concern 3: track page view once on mount
  useEffect(() => {
    analytics.page('dashboard');
  }, []);
}
```

Combining these into one effect creates coupling: if `userId` changes, the theme effect re-runs unnecessarily, and the analytics event fires again.

### Order of execution

Multiple effects in the same component run in declaration order, after every render where their respective dependencies changed.

```jsx
useEffect(() => { console.log('effect 1'); }, [a]);
useEffect(() => { console.log('effect 2'); }, [b]);

// If both a and b change in the same render, output is:
// effect 1
// effect 2
```

### Cleanup order for multiple effects

Before the new effects for a render run, React fires the cleanup functions of all effects whose dependencies changed, in declaration order.

```text
Render N+1 (both a and b changed):
  1. Cleanup of effect 1 (a changed)
  2. Cleanup of effect 2 (b changed)
  3. New effect 1
  4. New effect 2
```

---

## 16. Lifecycle Method Equivalents

### Mapping class lifecycle methods to useEffect

| Class lifecycle method | useEffect equivalent |
|---|---|
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate` (all updates) | `useEffect(() => { ... })` |
| `componentDidUpdate` (specific dep) | `useEffect(() => { ... }, [dep])` |
| `componentWillUnmount` | `return () => { ... }` inside any effect |
| `componentDidMount` + `componentWillUnmount` | `useEffect(() => { setup; return cleanup; }, [])` |
| `componentDidUpdate` (skip first render) | Requires a `useRef` flag (see below) |

### Skipping the first render in an effect

`useEffect` always runs on mount. There is no built-in equivalent of `componentDidUpdate` that skips mount. Use a ref as a flag.

```jsx
function Component({ value }) {
  const didMountRef = useRef(false);

  useEffect(() => {
    if (!didMountRef.current) {
      didMountRef.current = true;
      return; // skip on first render
    }
    console.log('value changed after initial mount:', value);
  }, [value]);
}
```

### Full class vs function comparison

```jsx
// Class component with setInterval
class Timer extends React.Component {
  state = { count: 0 };

  componentDidMount() {
    this.id = setInterval(this.tick, 1000);
  }
  componentWillUnmount() {
    clearInterval(this.id);
  }
  tick = () => this.setState(s => ({ count: s.count + 1 }));

  render() { return <p>{this.state.count}</p>; }
}

// Equivalent function component
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => setCount(c => c + 1), 1000);
    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>;
}
```

---

## 17. Common Mistakes

### Mistake 1: Forgetting the dependency array

```jsx
// ❌ Wrong — runs after every single render
useEffect(() => {
  document.title = 'My App';
}); // missing []

// ✅ Correct — runs once
useEffect(() => {
  document.title = 'My App';
}, []);
```

### Mistake 2: Missing dependencies — stale closure

```jsx
// ❌ Wrong — count is used but not listed; closure is stale
useEffect(() => {
  if (count > 10) doSomething();
}, []); // ESLint: 'count' is missing from the dependency array

// ✅ Correct
useEffect(() => {
  if (count > 10) doSomething();
}, [count]);
```

### Mistake 3: Object or array created in render placed in deps

```jsx
// ❌ Wrong — filters is a new array reference on every render
useEffect(() => {
  search(filters);
}, [filters]); // runs every render

// ✅ Fix option A — destructure to primitive deps
useEffect(() => {
  search({ category: filters.category, sort: filters.sort });
}, [filters.category, filters.sort]);

// ✅ Fix option B — memoize the object
const stableFilters = useMemo(
  () => ({ category: filters.category, sort: filters.sort }),
  [filters.category, filters.sort]
);
useEffect(() => { search(stableFilters); }, [stableFilters]);
```

### Mistake 4: Async callback directly

```jsx
// ❌ Wrong
useEffect(async () => {
  const data = await fetch('/api').then(r => r.json());
  setData(data);
}, []);

// ✅ Correct
useEffect(() => {
  async function load() {
    const data = await fetch('/api').then(r => r.json());
    setData(data);
  }
  load();
}, []);
```

### Mistake 5: Not cleaning up subscriptions

```jsx
// ❌ Wrong — subscription leaks on unmount
useEffect(() => {
  const unsub = store.subscribe(update);
}, []);

// ✅ Correct
useEffect(() => {
  const unsub = store.subscribe(update);
  return () => unsub();
}, []);
```

### Mistake 6: Computing derived state in an effect

```jsx
// ❌ Wrong — causes extra render; introduces timing issues
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(`${first} ${last}`);
}, [first, last]);

// ✅ Correct — compute during render, zero extra renders
const fullName = `${first} ${last}`;
```

---

## 18. Best Practices

### 1. One effect, one concern

Split effects by responsibility, not by timing. Combining unrelated side effects into one effect creates coupling and incorrect dependency arrays.

```jsx
// ❌ Wrong — mixed concerns; analytics re-fires on userId change
useEffect(() => {
  fetchUser(userId);
  analytics.track('page_view');
}, [userId]);

// ✅ Correct — separate concerns with separate effects
useEffect(() => { fetchUser(userId); },             [userId]);
useEffect(() => { analytics.track('page_view'); },  []);
```

### 2. Always clean up every resource the effect acquires

Any resource an effect opens (network connection, timer, listener, subscription) must be released in the cleanup. Treat every useEffect as a resource scope that must close.

### 3. Enable react-hooks/exhaustive-deps and respect it

The ESLint rule `react-hooks/exhaustive-deps` catches missing dependencies. Do not suppress it with a comment unless you have a documented reason. When the rule asks you to add a dependency, the correct fix is usually to restructure the code — not to silence the warning.

### 4. Extract complex effects into custom hooks

When an effect grows beyond a few lines, it is a strong signal to extract it. Custom hooks make the logic reusable, independently testable, and keep the component tree readable.

```jsx
// Component
function ChatRoom({ roomId }) {
  useChatRoom(roomId);
  return <div>...</div>;
}

// Custom hook
function useChatRoom(roomId) {
  useEffect(() => {
    const connection = createConnection(roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [roomId]);
}
```

### 5. Prefer functional state updates inside effects

Using `setState(prev => ...)` removes the need to list state variables in the dependency array, reduces stale closure risk, and keeps effects lean.

### 6. Never use effects for derived state

If a value can be computed from props or state during render, compute it during render. Using an effect for derivation causes an extra render cycle and introduces latency.

### 7. Test cleanup with React Strict Mode

Strict Mode's double-mount in development is a free test of your cleanup logic. If the component behaves correctly under double-mount, it will behave correctly in production. Fix the cleanup rather than disabling Strict Mode.

### 8. Use AbortController for all fetch requests inside effects

Cancelling the request on cleanup prevents race conditions, avoids state updates on unmounted components, and frees browser resources immediately when the user navigates away.

---
