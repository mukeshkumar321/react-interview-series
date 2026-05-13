# useState

## Table of Contents

- [1. What is useState?](#1-what-is-usestate)
- [2. How useState Works Internally](#2-how-usestate-works-internally)
- [3. Initial State](#3-initial-state)
- [4. Reading State](#4-reading-state)
- [5. Updating State](#5-updating-state)
- [6. State Updates are Asynchronous](#6-state-updates-are-asynchronous)
- [7. Functional Updates](#7-functional-updates)
- [8. Batching in React 18](#8-batching-in-react-18)
- [9. State with Objects](#9-state-with-objects)
- [10. State with Arrays](#10-state-with-arrays)
- [11. Stale Closures](#11-stale-closures)
- [12. Multiple State Variables vs One Object](#12-multiple-state-variables-vs-one-object)
- [13. Derived State](#13-derived-state)
- [14. State Initialization from Props Anti-pattern](#14-state-initialization-from-props-anti-pattern)
- [15. Lifting State Up](#15-lifting-state-up)
- [16. useState vs useReducer](#16-usestate-vs-usereducer)
- [17. useState and StrictMode](#17-usestate-and-strictmode)
- [18. Performance: When Does useState Trigger Re-render?](#18-performance-when-does-usestate-trigger-re-render)
- [19. Common Mistakes](#19-common-mistakes)
- [20. Best Practices](#20-best-practices)

---

## 1. What is useState?

`useState` is a React Hook that adds a single piece of local state to a functional component. When the state value changes, React schedules a re-render of the component with the new value.

It is the most fundamental hook — the entry point for any reactive data in a functional component.

---

### Syntax

```jsx
const [state, setState] = useState(initialValue);
```

- `state` — the current state value for this render
- `setState` — a function that updates the state and schedules a re-render
- `initialValue` — the value used on the very first render only

---

### Basic Example

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(count - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

---

### What useState Returns

`useState` returns a two-element array. Destructuring this array is the standard convention:

```jsx
const [value, setValue] = useState(0);
```

The names `value` and `setValue` are arbitrary — you choose them. Convention is `[name, setName]`.

---

### Multiple useState Calls

A component can call `useState` as many times as needed. Each call is independent.

```jsx
function Form() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [age, setAge] = useState(0);

  return (
    <form>
      <input value={name} onChange={e => setName(e.target.value)} />
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <input value={age} onChange={e => setAge(Number(e.target.value))} />
    </form>
  );
}
```

---

## 2. How useState Works Internally

Understanding how React stores state internally explains many of its behaviors, including why hooks must be called in the same order every render.

---

### Fiber Node and Hook List

React represents each component instance as a Fiber node. Each Fiber node contains a linked list of hook objects — one for each hook call in the component.

```text
Component Fiber Node
  ↓
memoizedState (hook linked list)
  ↓
Hook 0: useState  → { memoizedState: 0,    queue: { pending: null } }
  ↓
Hook 1: useState  → { memoizedState: "",   queue: { pending: null } }
  ↓
Hook 2: useEffect → { memoizedState: ...,  deps: [] }
```

---

### Call Order Dependency

React does not track hooks by name or variable. It tracks them by their position in the call order. The first `useState` call always reads from slot 0, the second from slot 1, and so on.

```jsx
function App() {
  const [a] = useState(1); // slot 0
  const [b] = useState(2); // slot 1
  const [c] = useState(3); // slot 2
}
```

On re-render, React walks the same list in the same order. This is why hooks called conditionally or inside loops break the system.

---

### State Update Queue

When `setState` is called, React does not immediately update the state. Instead, it enqueues an update on the hook's `queue` and schedules a re-render.

```text
setState(newValue) called
        ↓
Update added to hook's queue
        ↓
React schedules re-render
        ↓
Re-render begins
        ↓
React processes the update queue
        ↓
New state value computed
        ↓
Component renders with new state
```

---

### useState on Re-render

On the first render, React uses `initialValue`. On every subsequent render, React ignores `initialValue` and reads the current value from the hook's stored state.

```jsx
// initialValue is only used on the FIRST render
const [count, setCount] = useState(expensiveValue); // expensive computation runs every render!
```

This is why lazy initialization exists — see Section 3.

---

## 3. Initial State

The `initialValue` argument to `useState` is used exactly once: on the first render.

---

### Direct Value

```jsx
const [count, setCount] = useState(0);
const [name, setName] = useState("Alice");
const [isOpen, setIsOpen] = useState(false);
const [items, setItems] = useState([]);
```

---

### Lazy Initialization

When the initial state requires an expensive computation, pass a function instead of a value. React calls this function only on the first render.

```jsx
// Wrong — runs on every render
const [data, setData] = useState(parseLocalStorage());

// Correct — runs only on first render
const [data, setData] = useState(() => parseLocalStorage());
```

---

### When to Use Lazy Initialization

Use it whenever the initial value is expensive to compute:

- Reading from `localStorage` or `sessionStorage`
- Parsing a large data structure
- Running complex calculations

```jsx
function TodoList() {
  const [todos, setTodos] = useState(() => {
    const saved = localStorage.getItem("todos");
    return saved ? JSON.parse(saved) : [];
  });

  // ...
}
```

---

### Lazy Initialization with Props

Props can be used inside the lazy initializer:

```jsx
function SearchInput({ defaultQuery }) {
  const [query, setQuery] = useState(() => defaultQuery.trim().toLowerCase());

  // ...
}
```

The function runs once. After that, `defaultQuery` changes are ignored — this is intentional when you only need the initial value.

---

## 4. Reading State

State values in React are snapshots. Every render has its own fixed copy of all state values.

---

### State as a Snapshot

When a component renders, React takes a snapshot of the current state and gives it to the component. The values inside that render do not change — even if `setState` is called, or if time passes.

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1); // schedules a re-render with count = 1
    setCount(count + 1); // also schedules with count = 1 (same snapshot)
    setCount(count + 1); // also schedules with count = 1 (same snapshot)
    // Result: count becomes 1, not 3
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

---

### State in Async Callbacks

The snapshot is captured at the time the component rendered. Even if an async operation runs later, the state variable it closes over reflects the value at render time — not the latest value.

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setTimeout(() => {
      // count here is from the render when handleClick was created
      console.log(count); // always logs the snapshot value, not the latest
    }, 3000);
  }

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Inc</button>
      <button onClick={handleClick}>Log</button>
    </>
  );
}
```

---

### Reading the Latest State

To always access the latest state value regardless of when a callback was created, use `useRef`:

```jsx
function App() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;
  });

  function handleClick() {
    setTimeout(() => {
      console.log(countRef.current); // always latest
    }, 3000);
  }
}
```

---

## 5. Updating State

Calling the setter function returned by `useState` updates the state value for the next render.

---

### Basic Update

```jsx
setCount(5);
setName("Bob");
setIsOpen(true);
```

---

### Update Based on Previous State

When the new state depends on the previous state, always use the functional update form:

```jsx
setCount(prev => prev + 1);
```

This ensures you always operate on the most recent state, even when updates are batched or stale closures are involved.

---

### setState Does Not Merge

Unlike `this.setState` in class components, the functional `setState` replaces the entire value. There is no automatic merging.

```jsx
// Wrong — loses name
setState({ age: 25 });

// Correct — spreads existing state
setState(prev => ({ ...prev, age: 25 }));
```

---

### When the Re-render Happens

React does not immediately re-render when `setState` is called. The update is queued and React processes it at the next opportunity — typically at the end of the current event handler.

```text
Event handler starts
        ↓
setState called → update queued
setState called → update queued
        ↓
Event handler ends
        ↓
React processes update queue (batching)
        ↓
Component re-renders once with final state
```

---

## 6. State Updates are Asynchronous

`setState` does not synchronously change the state variable. This surprises many developers coming from other frameworks.

---

### Why Updates are Async

React batches state updates to improve performance. Instead of re-rendering after each individual `setState` call, React waits for the current JavaScript execution context to complete, then processes all queued updates in a single re-render.

---

### Demonstrating Async Behavior

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    console.log(count); // still 0 — state has NOT changed yet
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

`count` will log `0` even after `setCount(1)`. The new value is only available in the next render.

---

### React 17 and Earlier — Batching in Event Handlers

Before React 18, batching only occurred inside React event handlers. Updates in `setTimeout`, `Promise` callbacks, or native DOM event handlers were not batched.

```jsx
// React 17 — NOT batched (causes 2 re-renders)
setTimeout(() => {
  setCount(c => c + 1); // re-render 1
  setName("Alice");     // re-render 2
}, 0);
```

---

### React 18 — Automatic Batching Everywhere

React 18 introduced automatic batching. All state updates are batched by default, including those inside `setTimeout`, Promises, and native events.

```jsx
// React 18 — batched (causes 1 re-render)
setTimeout(() => {
  setCount(c => c + 1); // batched
  setName("Alice");     // batched
  // one re-render happens here
}, 0);
```

---

### Opting Out of Batching

If you need synchronous DOM updates before batching completes, use `flushSync` from `react-dom`:

```jsx
import { flushSync } from "react-dom";

function handleClick() {
  flushSync(() => {
    setCount(c => c + 1); // re-renders immediately
  });
  // DOM is updated here

  flushSync(() => {
    setName("Alice"); // re-renders immediately
  });
}
```

---

## 7. Functional Updates

Functional updates pass a function to `setState` instead of a direct value. The function receives the most recent state and returns the new state.

---

### Syntax

```jsx
setState(prevState => newState);
```

---

### When Functional Updates are Required

Use functional updates when the new state is computed from the previous state, especially:

- When updates are batched
- Inside `useEffect`, `useCallback`, or `setTimeout` closures
- When chaining multiple updates

```jsx
// Wrong — stale closure, count is always the snapshot
setCount(count + 1);
setCount(count + 1);
// Result: count + 1 (not count + 2)

// Correct — always uses latest value
setCount(prev => prev + 1);
setCount(prev => prev + 1);
// Result: count + 2
```

---

### Functional Updates in useEffect

```jsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // correct — does not need count in deps
  }, 1000);

  return () => clearInterval(id);
}, []); // empty deps — no stale closure issue
```

---

### When Functional Updates are Optional

When the update does not depend on the previous state, both forms are equivalent:

```jsx
setIsOpen(true);           // fine
setIsOpen(prev => true);   // also fine, but unnecessary
```

---

### Example — Toggle Pattern

```jsx
function Toggle() {
  const [isOn, setIsOn] = useState(false);

  // Correct — always toggles from current value
  function toggle() {
    setIsOn(prev => !prev);
  }

  return <button onClick={toggle}>{isOn ? "ON" : "OFF"}</button>;
}
```

---

## 8. Batching in React 18

React 18 introduced automatic batching as a performance improvement. This means multiple state updates are grouped together and trigger only one re-render.

---

### Before React 18

Batching only happened inside React synthetic event handlers:

```jsx
// Batched (React 17+)
<button onClick={() => {
  setA(1);   // }
  setB(2);   // } one render
}}>
```

```jsx
// Not batched in React 17 — two re-renders
fetch("/api").then(() => {
  setA(1); // render 1
  setB(2); // render 2
});
```

---

### React 18 Automatic Batching

All updates are batched everywhere:

```jsx
// React 18 — all of these cause only ONE re-render

// In event handlers
handleClick = () => { setA(1); setB(2); };

// In setTimeout
setTimeout(() => { setA(1); setB(2); }, 0);

// In Promise .then()
fetch("/api").then(() => { setA(1); setB(2); });

// In native event listeners
element.addEventListener("click", () => { setA(1); setB(2); });
```

---

### Checking Batching Behavior

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  console.log("render");

  function handleClick() {
    setA(1);
    setB(2);
    // Only ONE "render" log — both updates are batched
  }

  return <button onClick={handleClick}>Click</button>;
}
```

---

### Opting Out with flushSync

```jsx
import { flushSync } from "react-dom";

function handleClick() {
  flushSync(() => {
    setA(1); // forces immediate re-render
  });

  flushSync(() => {
    setB(2); // forces another immediate re-render
  });
  // Two renders total
}
```

---

## 9. State with Objects

React state can hold objects, but you must never mutate state directly. You must create a new object with the updated values.

---

### Why Direct Mutation Fails

React uses `Object.is` to compare old and new state. If you mutate the existing object and pass the same reference, React sees no change and skips re-rendering.

```jsx
// Wrong — same reference, no re-render
const handleClick = () => {
  user.name = "Bob";
  setUser(user); // same object reference
};
```

---

### Correct Pattern — Spread Operator

```jsx
const [user, setUser] = useState({ name: "Alice", age: 25 });

// Correct — new object reference
function updateName(newName) {
  setUser(prev => ({ ...prev, name: newName }));
}

function updateAge(newAge) {
  setUser(prev => ({ ...prev, age: newAge }));
}
```

---

### Nested Objects

For nested objects, you must spread at every level that changes:

```jsx
const [profile, setProfile] = useState({
  name: "Alice",
  address: {
    city: "Mumbai",
    pin: "400001"
  }
});

// Correct — spread outer and inner object
function updateCity(city) {
  setProfile(prev => ({
    ...prev,
    address: {
      ...prev.address,
      city
    }
  }));
}
```

---

### Using Immer for Complex Objects

For deeply nested objects, consider using `immer` to write mutations that produce new references under the hood:

```jsx
import { useImmer } from "use-immer";

function App() {
  const [profile, updateProfile] = useImmer({
    name: "Alice",
    address: { city: "Mumbai" }
  });

  function updateCity(city) {
    updateProfile(draft => {
      draft.address.city = city; // looks like mutation but is safe
    });
  }
}
```

---

## 10. State with Arrays

Arrays in state must also be treated as immutable. Do not mutate array methods like `push`, `pop`, `splice`, or `sort` on the state array directly.

---

### Adding an Item

```jsx
const [items, setItems] = useState([]);

// Correct — new array with item added
function addItem(item) {
  setItems(prev => [...prev, item]);
}

// Also correct — new array with item at beginning
function addItemFirst(item) {
  setItems(prev => [item, ...prev]);
}
```

---

### Removing an Item

```jsx
// Correct — filter returns a new array
function removeItem(id) {
  setItems(prev => prev.filter(item => item.id !== id));
}
```

---

### Updating an Item

```jsx
// Correct — map returns a new array
function updateItem(id, newValue) {
  setItems(prev =>
    prev.map(item => item.id === id ? { ...item, ...newValue } : item)
  );
}
```

---

### Sorting or Reversing

```jsx
// Wrong — sort mutates in place
setItems(items.sort((a, b) => a.name.localeCompare(b.name)));

// Correct — spread first to create a new array, then sort
setItems([...items].sort((a, b) => a.name.localeCompare(b.name)));
```

---

### Immutable Array Operations Summary

| Operation | Wrong (Mutates) | Correct (Immutable) |
|---|---|---|
| Add | `arr.push(x)` | `[...arr, x]` |
| Remove | `arr.splice(i, 1)` | `arr.filter(fn)` |
| Update | `arr[i] = x` | `arr.map(fn)` |
| Sort | `arr.sort(fn)` | `[...arr].sort(fn)` |
| Reverse | `arr.reverse()` | `[...arr].reverse()` |

---

## 11. Stale Closures

A stale closure occurs when a function captures a state variable from an older render and continues using that outdated value, even though the state has since changed.

---

### Definition

In JavaScript, closures capture variables by reference to the outer scope. In React functional components, each render creates new scope-level variables. A function defined during render A will always see the state values from render A, not from render B.

---

### Example — setTimeout Stale Closure

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setTimeout(() => {
      // This closure captured count = 0 (the value at click time)
      console.log(count); // always 0, even if count has changed
      setCount(count + 1); // adds 1 to the stale snapshot, not the latest value
    }, 3000);
  }

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>Inc</button>
      <button onClick={handleClick}>Log + Inc after 3s</button>
    </>
  );
}
```

---

### Fix 1 — Functional Update

Use functional updates when the new state depends on the previous state. The updater function always receives the most recent value:

```jsx
function handleClick() {
  setTimeout(() => {
    setCount(prev => prev + 1); // always uses latest count
  }, 3000);
}
```

This works because React internally passes the current state to the updater function at the time it runs — not at the time the closure was created.

---

### Fix 2 — useRef for Latest Value

When you only need to read the latest state (not update it), use a ref synchronized with the state:

```jsx
function App() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count; // always stays up-to-date
  });

  function handleClick() {
    setTimeout(() => {
      console.log(countRef.current); // always latest value
    }, 3000);
  }
}
```

---

### Stale Closure in useEffect

```jsx
// Wrong — count is stale inside the interval
useEffect(() => {
  const id = setInterval(() => {
    setCount(count + 1); // count is always 0
  }, 1000);
  return () => clearInterval(id);
}, []); // count not listed as dep

// Correct — functional update avoids stale closure
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1); // always current
  }, 1000);
  return () => clearInterval(id);
}, []);
```

---

## 12. Multiple State Variables vs One Object

Deciding whether to store state as multiple separate variables or a single object is a design choice with clear guidelines.

---

### Multiple State Variables

Best when state values change independently and represent unrelated concerns:

```jsx
function ProfilePage() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [bio, setBio] = useState("");
}
```

Each update only touches the relevant value. No spreading needed.

---

### One State Object

Best when multiple values always change together or represent a cohesive data structure:

```jsx
function ProfilePage() {
  const [profile, setProfile] = useState({
    name: "",
    email: "",
    bio: ""
  });

  function updateField(field, value) {
    setProfile(prev => ({ ...prev, [field]: value }));
  }
}
```

---

### Guidelines

| Situation | Prefer |
|---|---|
| Values change independently | Separate variables |
| Values represent a single entity | Single object |
| Values always update together | Single object |
| Using `useReducer` patterns | Single object with reducer |
| Simple primitives | Separate variables |

---

### Anti-Pattern — One Giant Object for Everything

```jsx
// Avoid — unrelated data forced together
const [state, setState] = useState({
  count: 0,
  modalOpen: false,
  userEmail: "",
  themeIsDark: false,
  selectedItemId: null
});
```

When `count` changes, all state fields re-render together unnecessarily. Split these into separate concerns.

---

## 13. Derived State

Derived state is data that can be computed directly from existing state or props. Adding a separate state variable for something that can be derived is an anti-pattern.

---

### Example — Anti-Pattern

```jsx
// Wrong — doubledCount can be derived from count
function App() {
  const [count, setCount] = useState(0);
  const [doubledCount, setDoubledCount] = useState(0); // redundant

  function increment() {
    const newCount = count + 1;
    setCount(newCount);
    setDoubledCount(newCount * 2); // must stay in sync manually
  }
}
```

---

### Correct Pattern — Compute During Render

```jsx
// Correct — derive during render
function App() {
  const [count, setCount] = useState(0);
  const doubledCount = count * 2; // always in sync, no extra state

  return (
    <>
      <p>Count: {count}</p>
      <p>Doubled: {doubledCount}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </>
  );
}
```

---

### When Derivation is Expensive

If the derived value is computationally expensive, memoize it with `useMemo`:

```jsx
const [items, setItems] = useState([...largeList]);
const [filter, setFilter] = useState("");

// Derived but memoized — only recomputes when deps change
const filteredItems = useMemo(
  () => items.filter(i => i.name.includes(filter)),
  [items, filter]
);
```

---

### Ask Before Adding State

Before adding a new state variable, ask: "Can this value be computed from existing state or props?" If yes, compute it instead of storing it.

---

## 14. State Initialization from Props Anti-pattern

Initializing state from a prop value creates a one-time copy that will become stale if the prop changes later.

---

### The Anti-Pattern

```jsx
// Wrong — state is initialized from prop but never updates when prop changes
function UserCard({ initialName }) {
  const [name, setName] = useState(initialName);
  // If parent passes a new initialName, name does not change
}
```

---

### Why It Is Wrong

`useState(initialValue)` only uses `initialValue` on the first render. On subsequent renders, React ignores the argument. If the parent passes a new `initialName`, the component's `name` state remains at the original value.

---

### When It Is Acceptable

This pattern is intentional when you need an editable local copy that starts from a prop value but diverges independently:

```jsx
// Acceptable — editing a form that starts with server data
function EditForm({ serverData }) {
  const [draft, setDraft] = useState(serverData); // intentional one-time copy
  // User can edit draft without affecting parent
}
```

---

### The Correct Alternative — Controlled Component

If the parent should control the value, remove the state entirely and use the prop directly:

```jsx
// Correct — fully controlled by parent
function UserCard({ name, onNameChange }) {
  return <input value={name} onChange={e => onNameChange(e.target.value)} />;
}
```

---

### Using a Key to Reset State

When you need child state to reset when a prop changes, use the `key` prop:

```jsx
// Parent resets child state by changing the key
<EditForm key={userId} serverData={serverData} />
```

When `key` changes, React unmounts and remounts the component, reinitializing all state.

---

## 15. Lifting State Up

When multiple components need to share or synchronize state, move the state up to their closest common ancestor. This is called lifting state up.

---

### When to Lift State

- Two sibling components need to reflect the same data
- A parent needs to know about a child's current value
- Two components need to stay synchronized

---

### Example

```jsx
// Before — each child has its own temperature, they can't sync
function TemperatureInput() {
  const [temp, setTemp] = useState(0);
  return <input value={temp} onChange={e => setTemp(e.target.value)} />;
}

function App() {
  return (
    <>
      <TemperatureInput /> {/* Celsius */}
      <TemperatureInput /> {/* Fahrenheit */}
    </>
  );
}
```

```jsx
// After — state lifted to parent, children are controlled
function App() {
  const [celsius, setCelsius] = useState(0);
  const fahrenheit = celsius * 9 / 5 + 32;

  return (
    <>
      <input
        value={celsius}
        onChange={e => setCelsius(Number(e.target.value))}
      />
      <input
        value={fahrenheit}
        onChange={e => setCelsius((Number(e.target.value) - 32) * 5 / 9)}
      />
    </>
  );
}
```

---

### Trade-off

Lifting state up can cause unnecessary re-renders of parent and sibling components. For large applications, consider splitting state into smaller contexts or using state management libraries.

---

## 16. useState vs useReducer

Both hooks manage component state. The choice between them depends on the complexity of the state logic.

---

### Comparison Table

| Scenario | Prefer |
|---|---|
| Simple boolean, string, or number | `useState` |
| State with two or three values | `useState` (separate variables) |
| Multiple related values updated together | `useReducer` |
| Next state depends on previous in complex ways | `useReducer` |
| Many distinct named actions | `useReducer` |
| Logic needs to be tested independently | `useReducer` |
| State transitions need to be logged or debugged | `useReducer` |

---

### useState — Simple Case

```jsx
function Toggle() {
  const [isOn, setIsOn] = useState(false);
  return <button onClick={() => setIsOn(v => !v)}>{isOn ? "ON" : "OFF"}</button>;
}
```

---

### useReducer — Complex Case

```jsx
const initialState = { count: 0, step: 1, max: 10 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return {
        ...state,
        count: Math.min(state.count + state.step, state.max)
      };
    case "setStep":
      return { ...state, step: action.payload };
    case "reset":
      return initialState;
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "setStep", payload: 5 })}>
        Step 5
      </button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </>
  );
}
```

---

### The Threshold

A common rule of thumb: when you have 3 or more related state variables that update in coordinated ways, consider switching to `useReducer`.

---

## 17. useState and StrictMode

React `StrictMode` intentionally double-invokes certain functions in development to help detect impure logic and side effects.

---

### What Gets Double-Invoked

In development with `React.StrictMode`:

- Component function bodies
- State initializer functions passed to `useState`
- Reducer functions passed to `useReducer`
- Functions passed to `useMemo` and `useCallback`

---

### Example

```jsx
function App() {
  const [count, setCount] = useState(() => {
    console.log("initializer"); // logs twice in development
    return 0;
  });

  console.log("render"); // logs twice in development

  return <h1>{count}</h1>;
}
```

---

### Why React Does This

Double invocation is how React verifies that these functions are pure (no side effects). If a function is pure, calling it twice produces the same result both times without any observable difference.

If you see unexpected behavior from double invocation, you have a side effect in a place that should be pure.

---

### Production Behavior

Double invocation only happens in development. In production, React does not double-invoke anything.

---

## 18. Performance: When Does useState Trigger Re-render?

Not every `setState` call causes a re-render. React uses `Object.is` to determine whether the new state is different from the current state.

---

### Object.is Comparison

`Object.is` is similar to `===` with two exceptions:

```js
Object.is(NaN, NaN) // true  (unlike ===)
Object.is(+0, -0)   // false (unlike ===)
```

For all practical purposes in state management, `Object.is` behaves like strict equality.

---

### Same Value — No Re-render

```jsx
const [count, setCount] = useState(0);

setCount(0);    // Object.is(0, 0) = true  → no re-render
setCount(0.0);  // Object.is(0, 0) = true  → no re-render

const [user, setUser] = useState({ name: "Alice" });
setUser(user);  // Object.is(user, user) = true → no re-render (same ref)
```

---

### New Value — Re-render

```jsx
setCount(1);                    // Object.is(0, 1) = false → re-render
setUser({ ...user, age: 25 });  // new object reference → re-render
setItems([...items]);           // new array reference → re-render
```

---

### Bailout

When React bails out (skips re-render), children also do not re-render. The component function itself may still be called once to confirm the bailout, but no UI update occurs.

---

### Optimization Pattern

```jsx
const [theme, setTheme] = useState("light");

// No re-render if theme is already "light"
function switchToLight() {
  setTheme("light"); // bails out if already "light"
}
```

---

## 19. Common Mistakes

---

### Mutating Objects Directly

❌ Wrong

```jsx
const [user, setUser] = useState({ name: "Alice" });

function update() {
  user.name = "Bob"; // mutates state directly
  setUser(user);     // same reference — React may skip re-render
}
```

✅ Correct

```jsx
function update() {
  setUser(prev => ({ ...prev, name: "Bob" }));
}
```

---

### Not Using Functional Update When Needed

❌ Wrong

```jsx
function App() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount(count + 1);
    setCount(count + 1); // both use stale count — result: count + 1
    setCount(count + 1); // not count + 3
  }
}
```

✅ Correct

```jsx
function increment() {
  setCount(prev => prev + 1);
  setCount(prev => prev + 1);
  setCount(prev => prev + 1); // result: count + 3
}
```

---

### Initializing State from a Prop That Changes

❌ Wrong

```jsx
function Input({ defaultValue }) {
  const [value, setValue] = useState(defaultValue); // stale if prop changes
}
```

✅ Correct (if controlled)

```jsx
function Input({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}
```

---

### Calling setState During Render Without a Guard

❌ Wrong

```jsx
function App() {
  const [count, setCount] = useState(0);
  setCount(1); // called unconditionally during render → infinite loop
  return <h1>{count}</h1>;
}
```

✅ Correct (if derivation is needed, just compute it)

```jsx
function App() {
  const [count, setCount] = useState(0);
  const display = count + 1; // derive instead of storing
  return <h1>{display}</h1>;
}
```

---

### Forgetting That State is Per Component Instance

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

function App() {
  return (
    <>
      <Counter /> {/* count = 0 */}
      <Counter /> {/* count = 0 — independent */}
      <Counter /> {/* count = 0 — independent */}
    </>
  );
}
```

Each `<Counter />` has its own state. They do not share a `count` variable.

---

## 20. Best Practices

---

### Use Descriptive Names

```jsx
// Poor
const [x, setX] = useState(false);

// Clear
const [isMenuOpen, setIsMenuOpen] = useState(false);
const [isLoading, setIsLoading] = useState(false);
const [hasError, setHasError] = useState(false);
```

---

### Prefer Separate Variables Over One Object

For unrelated state, use separate `useState` calls. This makes updates simpler and avoids unintentional coupling.

---

### Use Lazy Initialization for Expensive Initial Values

```jsx
const [config, setConfig] = useState(() => JSON.parse(localStorage.getItem("config") || "{}"));
```

---

### Always Use Functional Updates When New State Depends on Previous State

```jsx
setCount(prev => prev + 1);
setItems(prev => [...prev, newItem]);
setUser(prev => ({ ...prev, name: newName }));
```

---

### Derive Values Instead of Duplicating State

If a value can be computed from existing state, compute it during render or memoize it — do not store it in separate state.

---

### Use useReducer When Logic Becomes Complex

When you have more than three closely related state variables or complex conditional update logic, refactor to `useReducer`.

---

### Do Not Initialize State from Props Unless Intentional

If you want a component to stay synchronized with a prop, use the prop directly. If you need a local editable copy that starts from a prop, document the intent clearly and use a `key` reset pattern when needed.
