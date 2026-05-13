# React useRef Hook

## Table of Contents

1. [What is useRef](#1-what-is-useref)
2. [Syntax and Basic Usage](#2-syntax-and-basic-usage)
3. [Use Case 1 — DOM References](#3-use-case-1--dom-references)
4. [Use Case 2 — Mutable Values (No Re-render)](#4-use-case-2--mutable-values-no-re-render)
5. [useRef vs useState](#5-useref-vs-usestate)
6. [Storing Previous Value Pattern](#6-storing-previous-value-pattern)
7. [useRef for Stale Closure Fix](#7-useref-for-stale-closure-fix)
8. [Storing Timer References](#8-storing-timer-references)
9. [forwardRef](#9-forwardref)
10. [useImperativeHandle](#10-useimperativehandle)
11. [Null Check Pattern](#11-null-check-pattern)
12. [useRef vs createRef](#12-useref-vs-createref)
13. [Common Mistakes](#13-common-mistakes)
14. [Best Practices](#14-best-practices)

---

## 1. What is useRef

`useRef` is a React Hook that returns a **mutable ref object** — a plain JavaScript object with a single property: `.current`. This object **persists for the full lifetime of the component** and does **not** trigger a re-render when its `.current` value is mutated.

### Core Characteristics

- Returns `{ current: initialValue }` — a stable object reference across all renders
- Mutating `ref.current` **does not** schedule a re-render
- The ref object itself (the `{ current }` container) never changes between renders — only `.current` changes
- Lives as long as the component lives; destroyed on unmount
- `initialValue` is only used on the first render and ignored on all subsequent renders

### Two Primary Use Cases

```text
useRef Use Cases
      ↓
┌─────────────────────────────────┐
│                                 │
│  1. DOM Access                  │
│     → attach via ref prop       │
│     → ref.current = DOM node    │
│                                 │
│  2. Mutable Value Storage       │
│     → no re-render on change    │
│     → persists across renders   │
│                                 │
└─────────────────────────────────┘
```

### Mental Model

Think of `useRef` as a **box** sitting outside the render cycle. You can put anything in the box and take it out without React knowing or caring. React only re-renders when state changes — not when the box contents change.

```jsx
const ref = useRef(0);
// ref is { current: 0 }
// ref.current = 99 → nothing re-renders, React is unaware
```

### What useRef Is NOT

- Not a replacement for `useState` when you need the value displayed in the UI
- Not a way to share state between sibling components (use context or state lifting)
- Not the same as a class instance variable in class components, though the intent is similar

---

## 2. Syntax and Basic Usage

### Basic Syntax

```jsx
import { useRef } from 'react';

const ref = useRef(initialValue);
```

- `initialValue` — any value: number, string, null, object, function, DOM node reference, etc.
- Only used on the **first render**; completely ignored on subsequent renders
- Returns `{ current: initialValue }`

### Accessing and Mutating the Value

```jsx
const ref = useRef(42);

// Read
console.log(ref.current); // 42

// Mutate
ref.current = 100;
console.log(ref.current); // 100
// No re-render occurred
```

### Attaching to a DOM Element

```jsx
const inputRef = useRef(null);

return <input ref={inputRef} />;
// After mount: inputRef.current === <input> DOM element
```

React sets `ref.current` to the DOM element **after** the DOM has been committed to the screen, and resets it back to `null` when the element is removed from the DOM.

### Render Lifecycle with useRef

```text
Component renders
      ↓
React evaluates JSX
(ref.current is still null for DOM refs at this point)
      ↓
DOM is committed (mounted or updated)
      ↓
React assigns: ref.current = DOM node   ← ref populated HERE
      ↓
useLayoutEffect runs (ref.current is available)
      ↓
Browser paints
      ↓
useEffect runs (ref.current is available here too)
```

> Key rule: Never read a DOM ref during the render phase. It is `null` until after the DOM commit.

---

## 3. Use Case 1 — DOM References

### Attaching Refs to DOM Elements

The `ref` prop is a special reserved prop in React. When you pass a ref object to the `ref` prop of a DOM element, React populates `ref.current` with the actual DOM node after mount.

```jsx
function TextInput() {
  const inputRef = useRef(null);

  const focusInput = () => {
    inputRef.current.focus(); // direct DOM method call
  };

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}
```

### Common DOM Operations via useRef

| Operation | Code |
|-----------|------|
| Focus an input | `ref.current.focus()` |
| Blur an input | `ref.current.blur()` |
| Scroll into view | `ref.current.scrollIntoView({ behavior: 'smooth' })` |
| Measure dimensions | `ref.current.getBoundingClientRect()` |
| Control a video/audio | `ref.current.play()` / `ref.current.pause()` |
| Read scroll position | `ref.current.scrollTop` |
| Set scroll position | `ref.current.scrollTop = 0` |
| Read element width | `ref.current.offsetWidth` |
| Programmatic click | `ref.current.click()` |

### Measuring Element Dimensions

```jsx
function MeasureBox() {
  const boxRef = useRef(null);
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    if (boxRef.current) {
      const { width, height } = boxRef.current.getBoundingClientRect();
      setSize({ width, height });
    }
  }, []);

  return (
    <div ref={boxRef} style={{ padding: '20px', background: 'lightblue' }}>
      Width: {size.width}px, Height: {size.height}px
    </div>
  );
}
```

### Auto-scroll to Bottom (Chat UI Pattern)

```jsx
function ChatList({ messages }) {
  const bottomRef = useRef(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  return (
    <div className="chat-container">
      {messages.map((msg) => (
        <p key={msg.id}>{msg.text}</p>
      ))}
      <div ref={bottomRef} />
    </div>
  );
}
```

### Refs on Custom Components — The Problem

❌ Wrong — This does NOT work:

```jsx
function MyInput({ placeholder }) {
  return <input placeholder={placeholder} />;
}

function Parent() {
  const ref = useRef(null);
  return <MyInput ref={ref} />;
  // ref.current will be null — React logs a warning
}
```

React cannot pass `ref` as a regular prop to a custom functional component. `ref` is a special internal attribute intercepted by React before props are passed.

✅ Correct — Use `React.forwardRef`:

```jsx
const MyInput = React.forwardRef((props, ref) => (
  <input ref={ref} {...props} />
));

function Parent() {
  const ref = useRef(null);
  return <MyInput ref={ref} placeholder="Type here" />;
  // ref.current === <input> DOM node ✅
}
```

### Multiple Refs with Callback Ref Pattern

```jsx
function ImageGallery({ images }) {
  const itemRefs = useRef([]);

  const scrollToItem = (index) => {
    itemRefs.current[index]?.scrollIntoView({ behavior: 'smooth' });
  };

  return (
    <div>
      {images.map((img, i) => (
        <img
          key={img.id}
          src={img.src}
          ref={(el) => (itemRefs.current[i] = el)}
        />
      ))}
      <button onClick={() => scrollToItem(3)}>Jump to #4</button>
    </div>
  );
}
```

> Callback ref: React calls the function with the DOM node on mount and with `null` on unmount, giving you full control over ref assignment.

---

## 4. Use Case 2 — Mutable Values (No Re-render)

### The Core Idea

Some values need to be remembered across renders but should **not cause** a render when they change. `useRef` is the correct tool for this pattern.

### Which Values Belong Where

| Value Type | Tool | Reason |
|------------|------|--------|
| Displayed in JSX | `useState` | Must trigger re-render to update UI |
| Timer ID (interval, timeout) | `useRef` | No render needed |
| Previous render value | `useRef` | Updated after render |
| DOM node | `useRef` | Not rendered data |
| Boolean flag (e.g., isMounted) | `useRef` | No render needed |
| Animation frame ID | `useRef` | No render needed |
| External instance (WebSocket, player) | `useRef` | Not displayed in UI |

### Storing Timer IDs

```jsx
function Stopwatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
    if (intervalRef.current) return; // guard: prevent duplicate intervals
    intervalRef.current = setInterval(() => {
      setTime(t => t + 1);
    }, 1000);
  };

  const stop = () => {
    clearInterval(intervalRef.current);
    intervalRef.current = null;
  };

  return (
    <>
      <p>{time}s</p>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </>
  );
}
```

### Storing Render Count (Debugging Pattern)

```jsx
function RenderCounter() {
  const renderCount = useRef(0);
  renderCount.current += 1;

  return <p>Rendered {renderCount.current} times</p>;
}
```

> Incrementing during render does not cause another render. The displayed count updates because another re-render triggered by state change (or parent) already caused the component to run.

### isMounted Flag — Avoid setState on Unmounted Component

```jsx
function DataFetcher({ url }) {
  const [data, setData] = useState(null);
  const isMountedRef = useRef(true);

  useEffect(() => {
    isMountedRef.current = true;

    fetch(url)
      .then(res => res.json())
      .then(result => {
        if (isMountedRef.current) {
          setData(result); // only update state if component is still mounted
        }
      });

    return () => {
      isMountedRef.current = false;
    };
  }, [url]);

  return <div>{data ? JSON.stringify(data) : 'Loading...'}</div>;
}
```

### Storing External Instance

```jsx
function Chat({ roomId }) {
  const wsRef = useRef(null);

  useEffect(() => {
    wsRef.current = new WebSocket(`wss://chat.example.com/${roomId}`);

    wsRef.current.onmessage = (event) => {
      console.log('Message:', event.data);
    };

    return () => {
      wsRef.current?.close();
    };
  }, [roomId]);

  const sendMessage = (text) => {
    wsRef.current?.send(text);
  };

  return <button onClick={() => sendMessage('Hello')}>Send</button>;
}
```

---

## 5. useRef vs useState

### Comparison Table

| Feature | `useRef` | `useState` |
|---------|----------|------------|
| Returns | `{ current: value }` | `[value, setter]` |
| Re-render on change | No | Yes |
| Value persists across renders | Yes | Yes |
| Value stays current in JSX | No — stale between re-renders | Yes |
| Synchronous update | Yes (`ref.current = x`) | No — batched and async |
| Can be used as dependency in useEffect | Not recommended | Yes |
| Use for UI display | No | Yes |
| Use for DOM access | Yes | No |
| Use for timers / IDs | Yes | No |
| Use for previous values | Yes | No |

### Decision Flow

```text
Do you need to display the value in the UI?
      ↓
    YES → useState
      ↓
    NO → Does changing this value need to trigger a re-render?
           ↓
         YES → useState
           ↓
         NO → Is it a DOM element reference?
                   ↓
                 YES → useRef(null) + attach via ref prop
                   ↓
                  NO → useRef(initialValue) for mutable storage
```

### Code Contrast — ref for displayed value is wrong

❌ Wrong — UI will never update:

```jsx
function Counter() {
  const countRef = useRef(0);

  const increment = () => {
    countRef.current += 1;
    // No re-render scheduled → UI stays at 0 forever
  };

  return (
    <div>
      <p>Count: {countRef.current}</p> {/* Always renders 0 */}
      <button onClick={increment}>+</button>
    </div>
  );
}
```

✅ Correct — use state for anything displayed:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

### Using Both Together — Idiomatic Pattern

```jsx
function SmartTimer() {
  const [seconds, setSeconds] = useState(0);    // displayed → useState
  const intervalRef = useRef(null);             // timer ID → useRef
  const prevSecondsRef = useRef(0);             // previous value → useRef

  useEffect(() => {
    prevSecondsRef.current = seconds;
  });

  const start = () => {
    intervalRef.current = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);
  };

  return (
    <div>
      <p>Current: {seconds}s</p>
      <p>Previous: {prevSecondsRef.current}s</p>
      <button onClick={start}>Start</button>
    </div>
  );
}
```

---

## 6. Storing Previous Value Pattern

### The Pattern

A canonical React interview pattern: track the **previous** value of a state or prop using `useRef` and `useEffect`.

```jsx
function usePrevious(value) {
  const ref = useRef(undefined);

  useEffect(() => {
    ref.current = value;
  }); // no dependency array — runs after every single render

  return ref.current; // returns the value from the PREVIOUS render
}
```

### How It Works — Render Timeline

```text
Render 1 (count = 0):
  value passed in = 0
  ref.current = undefined  (initial state)
  usePrevious returns: undefined
  After render: effect runs → ref.current = 0

Render 2 (count = 1):
  value passed in = 1
  ref.current = 0  (set during the previous effect)
  usePrevious returns: 0     ← correct previous value ✅
  After render: effect runs → ref.current = 1

Render 3 (count = 2):
  value passed in = 2
  ref.current = 1
  usePrevious returns: 1     ← correct previous value ✅
```

### Full Example

```jsx
import { useState, useRef, useEffect } from 'react';

function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value;
  }); // deliberately no deps array

  return ref.current;
}

function PreviousCounter() {
  const [count, setCount] = useState(0);
  const previousCount = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {previousCount ?? 'none'}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

### Why useEffect and Not Inline Assignment?

```text
If you write ref.current = value directly in render:
  → ref.current is updated DURING the current render
  → Both current and previous display the same value
  → You lose the "one render behind" behaviour

useEffect runs AFTER the render completes:
  → During render N: ref.current still holds render N-1's value ← previous ✅
  → After render N: effect updates ref.current = render N's value
  → During render N+1: ref.current correctly holds render N's value
```

### Tracking Previous Props

```jsx
function UserCard({ userId }) {
  const prevUserId = usePrevious(userId);

  useEffect(() => {
    if (prevUserId !== undefined && prevUserId !== userId) {
      console.log(`User changed from ${prevUserId} to ${userId}`);
      // fetch new user data, reset local state, etc.
    }
  }, [userId, prevUserId]);

  return <div>User: {userId}</div>;
}
```

---

## 7. useRef for Stale Closure Fix

### The Stale Closure Problem

Event handlers and async callbacks **capture** values from their enclosing scope at the time they are **created**. If state updates after the closure was created, the closure still holds the old value.

```jsx
// Problem: stale closure in setInterval
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count); // Always logs 0 — stale closure!
    }, 1000);

    return () => clearInterval(id);
  }, []); // [] → effect runs only on mount → count captured as 0 forever

  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

### Why This Happens

```text
Render 1 (count = 0):
  useEffect with [] runs on mount
  setInterval callback closes over: count = 0
  count is frozen at 0 inside this callback

Render 2 (count = 1):
  React renders with count = 1 in the UI
  But the interval callback still captures count = 0
  → It logs 0 on every tick forever

This is the stale closure problem.
```

### The Fix Using useRef

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  // Keep ref synchronized with latest state after every render
  useEffect(() => {
    countRef.current = count;
  }, [count]);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(countRef.current); // Always reads the latest value ✅
    }, 1000);
    return () => clearInterval(id);
  }, []); // The interval is created once — reads through the ref at tick time

  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

### Why the Ref Fix Works

```text
The interval closure captures:
  → countRef  (the object — a STABLE reference, same object every render)
  → NOT countRef.current (the .current value inside)

Each render:
  → countRef.current is updated to the latest count via useEffect

Each interval tick:
  → countRef.current is READ at that exact moment
  → Always returns the latest value ✅

Key insight: Closing over the ref object (container) is safe.
Reading .current at runtime bypasses the stale closure
because .current was mutated after the closure was created.
```

### Alternative for State Updates — Functional Updater

If you only need to **update** state and don't need to read it, functional updater avoids stale closures entirely:

```jsx
const id = setInterval(() => {
  setCount(c => c + 1);
  // React provides the latest state as argument — no stale closure ✅
}, 1000);
```

For **reading** state inside async callbacks, `useRef` remains the correct solution.

### Event Handler Stale Closure Fix

```jsx
function Logger({ label }) {
  const labelRef = useRef(label);

  useEffect(() => {
    labelRef.current = label;
  }, [label]);

  useEffect(() => {
    const handler = () => {
      console.log('Label:', labelRef.current); // Always fresh ✅
    };
    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, []); // Handler registered once, reads latest label via ref

  return <div>{label}</div>;
}
```

---

## 8. Storing Timer References

### Why Store Timer IDs in Ref

Timer IDs from `setInterval` and `setTimeout` must be stored to cancel them later:

1. A local variable loses the ID when the component re-renders (new function scope)
2. Using `useState` for the ID triggers an unnecessary re-render
3. The timer ID is never displayed — it is not UI data

### setInterval with Start / Stop / Reset

```jsx
function Timer() {
  const [seconds, setSeconds] = useState(0);
  const [running, setRunning] = useState(false);
  const intervalRef = useRef(null);

  const startTimer = () => {
    if (intervalRef.current) return; // prevent duplicate intervals
    setRunning(true);
    intervalRef.current = setInterval(() => {
      setSeconds(s => s + 1);
    }, 1000);
  };

  const stopTimer = () => {
    clearInterval(intervalRef.current);
    intervalRef.current = null;
    setRunning(false);
  };

  const resetTimer = () => {
    stopTimer();
    setSeconds(0);
  };

  useEffect(() => {
    return () => clearInterval(intervalRef.current); // unmount cleanup
  }, []);

  return (
    <div>
      <p>{seconds}s</p>
      <button onClick={startTimer} disabled={running}>Start</button>
      <button onClick={stopTimer} disabled={!running}>Stop</button>
      <button onClick={resetTimer}>Reset</button>
    </div>
  );
}
```

### setTimeout — Debounce-like Pattern

```jsx
function SearchInput() {
  const [query, setQuery] = useState('');
  const timeoutRef = useRef(null);

  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);

    clearTimeout(timeoutRef.current);
    timeoutRef.current = setTimeout(() => {
      console.log('Searching for:', value);
    }, 500);
  };

  useEffect(() => {
    return () => clearTimeout(timeoutRef.current);
  }, []);

  return <input value={query} onChange={handleChange} placeholder="Search..." />;
}
```

### requestAnimationFrame

```jsx
function AnimatedBar() {
  const [width, setWidth] = useState(0);
  const rafRef = useRef(null);

  const animate = () => {
    setWidth(w => {
      if (w >= 100) {
        cancelAnimationFrame(rafRef.current);
        return 100;
      }
      return w + 1;
    });
    rafRef.current = requestAnimationFrame(animate);
  };

  const start = () => {
    rafRef.current = requestAnimationFrame(animate);
  };

  useEffect(() => {
    return () => cancelAnimationFrame(rafRef.current);
  }, []);

  return (
    <>
      <div style={{ width: `${width}%`, background: 'blue', height: 20 }} />
      <button onClick={start}>Animate</button>
    </>
  );
}
```

### Why Not useState for Timer IDs?

❌ Wrong:

```jsx
const [intervalId, setIntervalId] = useState(null);

const start = () => {
  const id = setInterval(() => setSeconds(s => s + 1), 1000);
  setIntervalId(id); // triggers a re-render — wasted and potentially buggy
};
```

✅ Correct:

```jsx
const intervalRef = useRef(null);

const start = () => {
  intervalRef.current = setInterval(() => setSeconds(s => s + 1), 1000);
  // No re-render triggered ✅
};
```

---

## 9. forwardRef

### The Problem

React does not allow passing `ref` to functional components as a regular prop. The `ref` attribute is intercepted by React before props are passed to the component function.

```jsx
function FancyInput(props) {
  return <input {...props} />;
}

// ❌ ref is NOT forwarded to the <input>
const ref = useRef(null);
<FancyInput ref={ref} />;
// ref.current stays null — React logs a warning
```

### React.forwardRef

`React.forwardRef` creates a component that receives a `ref` as its second argument and can forward it to a child DOM element or component.

```jsx
const FancyInput = React.forwardRef((props, ref) => {
  return <input ref={ref} className="fancy-input" {...props} />;
});

function Parent() {
  const inputRef = useRef(null);

  const handleFocus = () => {
    inputRef.current.focus();
  };

  return (
    <>
      <FancyInput ref={inputRef} placeholder="Fancy input" />
      <button onClick={handleFocus}>Focus</button>
    </>
  );
}
```

### Syntax Breakdown

```jsx
const Component = React.forwardRef(
  (props, ref) => {           // ref is always the second argument
    return <div ref={ref} {...props} />;
  }
);
```

| Parameter | Description |
|-----------|-------------|
| `props` | All regular props passed by the parent |
| `ref` | The ref forwarded from the parent — `null` if no ref was passed |

### Forwarding to a Specific Nested Element

```jsx
const Card = React.forwardRef(({ title, children }, ref) => (
  <div className="card">
    <h2>{title}</h2>
    <div className="card-body" ref={ref}>  {/* ref on the inner element */}
      {children}
    </div>
  </div>
));

function App() {
  const bodyRef = useRef(null);

  return (
    <Card title="My Card" ref={bodyRef}>
      <p>Content</p>
    </Card>
  );
}
```

### displayName for DevTools

```jsx
const FancyInput = React.forwardRef(function FancyInput(props, ref) {
  return <input ref={ref} {...props} />;
});

FancyInput.displayName = 'FancyInput'; // Shows correctly in React DevTools
```

### ref is NOT in props

```jsx
const Child = React.forwardRef((props, ref) => {
  console.log(props.ref); // undefined — ref is not a regular prop
  console.log(ref);       // the actual ref object ✅
  return <input ref={ref} />;
});
```

---

## 10. useImperativeHandle

### What It Does

`useImperativeHandle` lets you **customize the value** exposed when a parent accesses a child's ref. Instead of exposing the raw DOM node, you expose a custom object with only the methods you choose to make available.

### Syntax

```jsx
useImperativeHandle(ref, () => ({
  methodName: () => { /* implementation */ },
  anotherMethod: () => { /* implementation */ },
}), [deps]); // optional deps array — same semantics as useEffect
```

### Example — Custom Input with Restricted API

```jsx
const SmartInput = React.forwardRef((props, ref) => {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current.focus();
    },
    clear() {
      inputRef.current.value = '';
    },
    getValue() {
      return inputRef.current.value;
    },
  }));

  return <input ref={inputRef} {...props} />;
});

function Parent() {
  const inputRef = useRef(null);

  return (
    <>
      <SmartInput ref={inputRef} placeholder="Type here" />
      <button onClick={() => inputRef.current.focus()}>Focus</button>
      <button onClick={() => inputRef.current.clear()}>Clear</button>
      <button onClick={() => alert(inputRef.current.getValue())}>Get Value</button>
      {/* inputRef.current.style → undefined — intentionally not exposed */}
    </>
  );
}
```

### When to Use useImperativeHandle

- Restricting what a parent can imperatively do with a child
- Building component libraries where the internal DOM structure may change over time
- Exposing a semantic API (e.g., `open()` / `close()`) rather than a raw DOM node

### Comparison

| Aspect | Without `useImperativeHandle` | With `useImperativeHandle` |
|--------|-------------------------------|----------------------------|
| Exposes | Full DOM node (all native methods) | Only what you define |
| Encapsulation | Low | High |
| API surface | Implicit — all DOM methods | Explicit — your interface |
| Caller can do | `ref.current.style.color = 'red'` | Only `focus()`, `clear()`, etc. |

### Data Flow

```text
Parent:
  const ref = useRef(null);
  <SmartInput ref={ref} />
       ↓
React.forwardRef bridges the ref to the component
       ↓
useImperativeHandle assigns a custom object to ref.current
       ↓
Parent: ref.current.focus() → calls the custom focus function you defined
```

---

## 11. Null Check Pattern

### Why DOM Refs Start as Null

When you write `useRef(null)`, `.current` is `null` until the component mounts and React attaches the DOM node. The ref is always `null` during the render phase for DOM refs.

### Common Bug

❌ Wrong — accessing ref during render:

```jsx
function BadComponent() {
  const divRef = useRef(null);
  const width = divRef.current.offsetWidth; // TypeError: Cannot read properties of null

  return <div ref={divRef}>Hello</div>;
}
```

✅ Correct — access in useEffect or event handlers:

```jsx
function GoodComponent() {
  const divRef = useRef(null);
  const [width, setWidth] = useState(0);

  useEffect(() => {
    if (divRef.current) {
      setWidth(divRef.current.offsetWidth); // DOM is mounted here ✅
    }
  }, []);

  return <div ref={divRef}>Width: {width}px</div>;
}
```

### Optional Chaining for Safe Access

```jsx
const handleClick = () => {
  inputRef.current?.focus(); // No crash if ref is null
};
```

### TypeScript Pattern

```tsx
const inputRef = useRef<HTMLInputElement>(null);

useEffect(() => {
  if (inputRef.current) {
    inputRef.current.focus();
    // TypeScript narrows the type to HTMLInputElement inside this block
  }
}, []);
```

### Ref Lifecycle

```text
Before mount:
  ref.current = null   ← initial value set by useRef(null)

After mount:
  ref.current = <DOM element>   ← React sets this after DOM commit

After element is removed from DOM (unmount or conditional rendering):
  ref.current = null   ← React resets automatically
```

### Conditional Rendering Gotcha

```jsx
function Toggle() {
  const [show, setShow] = useState(true);
  const ref = useRef(null);

  const focusInput = () => {
    // ref.current is null when show is false — element not in DOM
    ref.current?.focus();
  };

  return (
    <>
      <button onClick={() => setShow(s => !s)}>Toggle</button>
      {show && <input ref={ref} />}
      <button onClick={focusInput}>Focus Input</button>
    </>
  );
}
```

---

## 12. useRef vs createRef

### Overview

| Feature | `useRef` | `createRef` |
|---------|----------|-------------|
| Designed for | Functional components | Class components |
| Creates new object per render | No — same object | Yes — new each render |
| Persists across renders | Yes | No (in functional components) |
| Correct usage | Functional components | Class component constructors |

### createRef in Class Components (Correct Usage)

```jsx
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.inputRef = React.createRef(); // created once in constructor — persists
  }

  componentDidMount() {
    this.inputRef.current.focus();
  }

  render() {
    return <input ref={this.inputRef} />;
  }
}
```

### The Problem with createRef in Functional Components

❌ Wrong:

```jsx
function BadComponent() {
  const ref = React.createRef(); // NEW ref object created on every render
  // ref.current is null at render time — previous DOM attachment is lost

  return <input ref={ref} />;
}
```

✅ Correct:

```jsx
function GoodComponent() {
  const ref = useRef(null); // same ref object persists across all renders

  return <input ref={ref} />;
}
```

### Why createRef Fails in Functional Components

```text
Render 1:
  createRef() → creates ref1 = { current: null }
  React commits DOM → ref1.current = <input>

Render 2 (any state update triggers a re-render):
  createRef() → creates ref2 = { current: null }  ← brand new object
  ref1 is now abandoned — ref1.current = <input> is orphaned
  ref2.current = null until React processes the ref on this render
  → You temporarily have no ref to the DOM node

useRef (same object every render):
  Render 1: ref = { current: null }
  After mount: ref.current = <input>
  Render 2: ref = { current: <input> }  ← same object, DOM node preserved ✅
```

---

## 13. Common Mistakes

### Mistake 1 — Reading ref.current During Render

❌ Wrong:

```jsx
function Component() {
  const ref = useRef(null);
  console.log(ref.current.clientHeight); // null during render → TypeError crash

  return <div ref={ref} />;
}
```

✅ Correct:

```jsx
function Component() {
  const ref = useRef(null);

  useEffect(() => {
    console.log(ref.current.clientHeight); // Available after mount ✅
  }, []);

  return <div ref={ref} />;
}
```

### Mistake 2 — Using ref When You Need Re-render

❌ Wrong — UI never updates:

```jsx
function Form() {
  const inputRef = useRef('');

  return (
    <div>
      <input onChange={e => { inputRef.current = e.target.value; }} />
      <p>You typed: {inputRef.current}</p> {/* Always empty — no re-render */}
    </div>
  );
}
```

✅ Correct:

```jsx
function Form() {
  const [value, setValue] = useState('');

  return (
    <div>
      <input onChange={e => setValue(e.target.value)} />
      <p>You typed: {value}</p>
    </div>
  );
}
```

### Mistake 3 — Forgetting That ref is { current: value }

❌ Wrong:

```jsx
const ref = useRef(0);
ref = 5;           // TypeError: assignment to constant — also wrong conceptually
console.log(ref);  // Logs { current: 0 }, not the primitive 0
```

✅ Correct:

```jsx
const ref = useRef(0);
ref.current = 5;          // mutate the .current property ✅
console.log(ref.current); // 5
```

### Mistake 4 — Not Cleaning Up Timers Stored in Ref

❌ Wrong — memory leak, interval runs after unmount:

```jsx
function Component() {
  const timerRef = useRef(null);

  const start = () => {
    timerRef.current = setInterval(doSomething, 1000);
  };

  return <button onClick={start}>Start</button>;
  // No cleanup → interval runs forever even after component unmounts
}
```

✅ Correct:

```jsx
function Component() {
  const timerRef = useRef(null);

  useEffect(() => {
    return () => {
      if (timerRef.current) {
        clearInterval(timerRef.current);
      }
    };
  }, []);

  const start = () => {
    timerRef.current = setInterval(doSomething, 1000);
  };

  return <button onClick={start}>Start</button>;
}
```

### Mistake 5 — Passing ref as a Prop Without forwardRef

❌ Wrong:

```jsx
function Child({ ref }) { // ref is undefined here
  return <input ref={ref} />;
}
```

✅ Correct:

```jsx
const Child = React.forwardRef((props, ref) => {
  return <input ref={ref} />;
});
```

### Mistake 6 — Side Effects in Render via ref

❌ Wrong — produces double side effects in StrictMode:

```jsx
function Component({ value }) {
  const logRef = useRef([]);
  logRef.current.push(value); // runs on every render including double renders

  return <div>{value}</div>;
}
```

✅ Correct:

```jsx
function Component({ value }) {
  const logRef = useRef([]);

  useEffect(() => {
    logRef.current.push(value);
  }, [value]);

  return <div>{value}</div>;
}
```

---

## 14. Best Practices

### When to Use useRef

| Scenario | Use useRef? |
|----------|-------------|
| Access a DOM element (focus, scroll, measure) | Yes |
| Store a timer ID (setInterval, setTimeout, rAF) | Yes |
| Track previous state or prop value | Yes |
| Fix stale closure in async / interval code | Yes |
| Store a mutable flag (isMounted, isRunning) | Yes |
| Store external instance (WebSocket, player) | Yes |
| Store a value you want to display | No — use useState |
| Cache an expensive computed value | No — use useMemo |
| Share state between sibling components | No — lift state or use context |

### Core Principles

1. **Never use refs to drive UI** — If the value needs to be rendered, use state. Refs entirely bypass React's rendering model.

2. **Prefer refs for imperative and external concerns** — DOM manipulation, timers, and external subscriptions are the canonical use cases.

3. **Always null-check before DOM operations** — Use `ref.current?.method()` or an explicit `if (ref.current)` guard before accessing DOM methods.

4. **Clean up timers in useEffect's cleanup** — Always return a cleanup function from `useEffect` when storing intervals or timeouts in a ref.

5. **Do not read DOM refs during render** — Refs are populated after mount. During render, DOM refs are always `null`.

6. **Use forwardRef for any public-facing component wrapper** — If your component wraps a DOM element and is intended to be used by other developers, use `forwardRef` to allow ref access.

7. **Use useImperativeHandle to restrict the API surface** — When building component libraries, expose only a defined interface rather than the full DOM node.

8. **Always prefer useRef over createRef in functional components** — `createRef` creates a new object on every render and is designed exclusively for class components.

### Summary

```text
useRef provides:
  ┌───────────────────────────────────────────────┐
  │  Persistence across renders                    │
  │  + No re-render on mutation                    │
  │  = Correct for non-UI mutable values           │
  └───────────────────────────────────────────────┘

useState provides:
  ┌───────────────────────────────────────────────┐
  │  Persistence across renders                    │
  │  + Re-render on update                         │
  │  = Correct for UI-driving values               │
  └───────────────────────────────────────────────┘
```
