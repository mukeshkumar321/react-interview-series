## 📚 useEffect — Tricky Output Questions

> These questions focus on predicting console output, identifying stale closures, understanding cleanup timing, async pitfalls, dependency array behavior, and React Strict Mode quirks. Each question reflects a real scenario you may encounter in a senior React interview.

---

## 1. Timing

### Q1

```jsx
function App() {
  console.log('A');

  useEffect(() => {
    console.log('B');
  });

  console.log('C');

  return <div />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
A
C
B
```

**Explanation:** `console.log('A')` and `console.log('C')` execute during the render phase, synchronously, before React commits to the DOM. `useEffect` fires asynchronously after the browser paints, so `B` always prints last. The render function runs top to bottom, so `A` comes before `C`.

</details>

---

### Q2

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('effect:', count);
  });

  console.log('render:', count);

  return <button onClick={() => setCount(1)}>Click</button>;
}
```

#### ❓ What is printed on initial render, and what is printed after the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
render: 0
effect: 0

// After click:
render: 1
effect: 1
```

**Explanation:** On every render cycle, the body of the component runs first (printing `render: <value>`), then React commits the DOM, the browser paints, and finally useEffect fires (printing `effect: <value>`). Because there is no dependency array, the effect runs after every render.

</details>

---

### Q3

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  useEffect(() => {
    console.log('effect with []');
  }, []);

  useEffect(() => {
    console.log('effect with [a]:', a);
  }, [a]);

  return (
    <div>
      <button onClick={() => setA(1)}>Set A</button>
      <button onClick={() => setB(1)}>Set B</button>
    </div>
  );
}
```

#### ❓ What is printed on mount? What is printed when "Set A" is clicked? When "Set B" is clicked?

<details>
<summary>✅ Answer</summary>

```txt
// On mount:
effect with []
effect with [a]: 0

// After clicking "Set A":
effect with [a]: 1

// After clicking "Set B":
(nothing — neither effect re-runs)
```

**Explanation:** On mount both effects run in declaration order. `[]` means run once. `[a]` means run when `a` changes. Changing `b` does not affect either effect's dependency array, so nothing runs.

</details>

---

### Q4

```jsx
function App() {
  useLayoutEffect(() => {
    console.log('layout effect');
  });

  useEffect(() => {
    console.log('effect');
  });

  console.log('render');

  return <div />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
render
layout effect
effect
```

**Explanation:** `render` executes during the render phase. `useLayoutEffect` fires synchronously after React commits the DOM but before the browser paints. `useEffect` fires asynchronously after the browser paints. So the order is: render → layout effect → (paint) → effect.

</details>

---

## 2. Cleanup

### Q5

```jsx
function App() {
  const [show, setShow] = useState(true);

  return show ? <Child /> : <p>Unmounted</p>;
}

function Child() {
  useEffect(() => {
    console.log('mount');
    return () => console.log('cleanup');
  }, []);

  return <p>Child</p>;
}
```

#### ❓ What is printed when `show` is set to `false`?

<details>
<summary>✅ Answer</summary>

```txt
// On initial render:
mount

// When show is set to false:
cleanup
```

**Explanation:** The cleanup function returned from the effect runs when the component unmounts. When `show` becomes `false`, `Child` is removed from the tree, triggering the cleanup.

</details>

---

### Q6

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('effect', count);
    return () => console.log('cleanup', count);
  }, [count]);

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ What is printed when the button is clicked twice (count goes 0 → 1 → 2)?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
effect 0

// After first click (count: 0 → 1):
cleanup 0
effect 1

// After second click (count: 1 → 2):
cleanup 1
effect 2
```

**Explanation:** The cleanup from the previous effect run always fires before the new effect fires. Each cleanup closes over the value of `count` at the time that effect was created — not the current value. So `cleanup 0` prints the old count, then `effect 1` prints the new count.

</details>

---

### Q7

```jsx
function App() {
  const [x, setX] = useState(0);

  useEffect(() => {
    console.log('setup');
    return () => console.log('cleanup');
  }, [x]);

  return <button onClick={() => setX(x)}>Same value</button>;
}
```

#### ❓ Does clicking the button trigger cleanup and re-run the effect?

<details>
<summary>✅ Answer</summary>

```txt
// On mount:
setup

// After clicking "Same value" (x stays 0):
(nothing)
```

**Explanation:** React uses `Object.is` to compare dependency values. `Object.is(0, 0)` is `true`, so React considers `x` unchanged. The cleanup and effect do not re-run. The effect only re-runs when a dependency's value actually changes.

</details>

---

### Q8

```jsx
function App() {
  useEffect(() => {
    console.log('effect');
    return () => console.log('cleanup');
  }, []);

  return <div />;
}
```

#### ❓ What is printed in development with React 18 Strict Mode?

<details>
<summary>✅ Answer</summary>

```txt
effect
cleanup
effect
```

**Explanation:** React 18 Strict Mode in development intentionally mounts, unmounts, and remounts every component to verify that cleanup functions work correctly. So the effect fires, then the cleanup fires (on the forced unmount), then the effect fires again (on the remount).

In production, this does not happen — you would see only `effect`.

</details>

---

### Q9

```jsx
useEffect(() => {
  return 42;
}, []);
```

#### ❓ Is this valid? What happens?

<details>
<summary>✅ Answer</summary>

```txt
React will log a warning in development:

Warning: An effect function must not return anything besides a function, 
which is used for clean-up. You returned: 42
```

**Explanation:** `useEffect` expects the setup function to return either nothing (`undefined`) or a cleanup function. Returning a non-function value (like `42`) is not valid. React will log a warning in development and ignore the return value. There is no runtime crash, but the non-function cleanup is silently discarded.

</details>

---

## 3. Stale Closures

### Q10

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1);
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>;
}
```

#### ❓ What does the counter display after 3 seconds?

<details>
<summary>✅ Answer</summary>

```txt
1
```

**Explanation:** The effect runs once on mount with `count = 0`. The interval callback closes over `count = 0`. Every second it calls `setCount(0 + 1)`. Count becomes `1`, but then `Object.is(1, 1)` is true — React sees no actual change and bails out. The counter is stuck at `1`.

The fix is to use the functional update form: `setCount(prev => prev + 1)`.

</details>

---

### Q11

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setTimeout(() => {
      console.log('count inside timeout:', count);
    }, 3000);
  }, []);

  return <button onClick={() => setCount(5)}>Set to 5</button>;
}
```

#### ❓ If the button is clicked immediately after mount, what does the timeout log after 3 seconds?

<details>
<summary>✅ Answer</summary>

```txt
count inside timeout: 0
```

**Explanation:** The effect runs once on mount, at which point `count = 0`. The timeout callback is a closure created at mount time capturing `count = 0`. Clicking the button updates the component's state and triggers a re-render, but the closure inside the already-scheduled timeout is frozen at the value it captured when created. It always logs `0`.

</details>

---

### Q12

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      setCount(prev => prev + 1); // functional update
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return <p>{count}</p>;
}
```

#### ❓ What does the counter display after 3 seconds?

<details>
<summary>✅ Answer</summary>

```txt
3
```

**Explanation:** The functional update form `setCount(prev => prev + 1)` receives the latest state value directly from React's state queue, not from the closure. The closure does not need to capture `count` at all. Every second, `prev` is the current count, so the counter correctly increments to 1, 2, 3, ...

</details>

---

### Q13

```jsx
function App() {
  const [value, setValue] = useState('initial');
  const valueRef = useRef(value);

  useEffect(() => {
    valueRef.current = value;
  }, [value]);

  useEffect(() => {
    setTimeout(() => {
      console.log('ref:', valueRef.current);
    }, 2000);
  }, []);

  return <button onClick={() => setValue('updated')}>Update</button>;
}
```

#### ❓ If the button is clicked within 2 seconds of mount, what does the timeout log?

<details>
<summary>✅ Answer</summary>

```txt
ref: updated
```

**Explanation:** `valueRef` is a mutable object. When `value` changes, the first effect updates `valueRef.current` to the new value. When the timeout fires at 2 seconds, it reads `valueRef.current` — which is a live reference to the mutable ref object, not a snapshot. So it reads `"updated"`, not `"initial"`. This is exactly the pattern used to get always-fresh values without adding to the dependency array.

</details>

---

### Q14

```jsx
function App() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount(count + 1);
  }

  useEffect(() => {
    document.addEventListener('click', increment);
    return () => document.removeEventListener('click', increment);
  }, []);

  return <p>{count}</p>;
}
```

#### ❓ What happens when the document is clicked multiple times?

<details>
<summary>✅ Answer</summary>

```txt
The counter increments to 1 on the first click, then gets stuck at 1 on all subsequent clicks.
```

**Explanation:** The effect runs once on mount. At that point `count = 0`, so `increment` is the version of the function that calls `setCount(0 + 1)`. Even though the component re-renders and creates a new `increment` function with the updated `count`, the event listener still holds a reference to the original `increment` (the one that always sets count to 1). This is a stale closure via event listener. Fix: add `count` or `increment` to the dependency array, or use `setCount(prev => prev + 1)`.

</details>

---

## 4. Dependency Array

### Q15

```jsx
function App() {
  const [count, setCount] = useState(0);

  const options = { threshold: count };

  useEffect(() => {
    console.log('effect ran');
  }, [options]);

  return <button onClick={() => setCount(0)}>Click</button>;
}
```

#### ❓ Does the effect re-run when the button is clicked? Why?

<details>
<summary>✅ Answer</summary>

```txt
Yes — the effect re-runs on every click even though count does not change.
```

**Explanation:** `options` is an object literal created inline inside the component body. Every render produces a new object with a new reference, even if the shape and values are identical. React's `Object.is({...}, {...})` is `false` because they are different objects in memory. So `options` is always "changed" as far as React is concerned, and the effect runs after every render.

Fix: move `options` inside the effect, or use `useMemo`, or pass `count` directly as a dependency.

</details>

---

### Q16

```jsx
function App() {
  const [x, setX] = useState(NaN);

  useEffect(() => {
    console.log('effect ran');
  }, [x]);

  return <button onClick={() => setX(NaN)}>Set NaN</button>;
}
```

#### ❓ Does the effect re-run when the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
No — the effect does NOT re-run after the first click.
```

**Explanation:** React uses `Object.is` for dependency comparison. Unlike `===`, `Object.is(NaN, NaN)` is `true`. So React correctly identifies that `x` has not changed between renders (NaN === NaN in Object.is terms) and skips re-running the effect.

</details>

---

### Q17

```jsx
function App() {
  const [count, setCount] = useState(0);

  const double = () => count * 2; // new function reference every render

  useEffect(() => {
    console.log(double());
  }, [double]); // double is in the deps array

  return <button onClick={() => setCount(c => c + 1)}>+1</button>;
}
```

#### ❓ How often does the effect run?

<details>
<summary>✅ Answer</summary>

```txt
After every single render, regardless of whether count changed.
```

**Explanation:** `double` is defined inline in the component body. Every render creates a new function object with a new reference. `Object.is(fn1, fn2)` is `false` even if both functions have identical bodies. So React sees `double` as changed every render and re-runs the effect every render.

Fix: wrap `double` in `useCallback` with `[count]` as deps, or move the function inside the effect.

</details>

---

### Q18

```jsx
function App({ id }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetchData(id).then(res => setData(res));
  }, []); // id is missing from deps

  return <p>{data?.name}</p>;
}
```

#### ❓ What is the bug? What happens when the `id` prop changes?

<details>
<summary>✅ Answer</summary>

```txt
When `id` changes, the effect does NOT re-run.
The component continues displaying data for the original id from mount.
```

**Explanation:** The empty dependency array `[]` means "run once on mount." At mount, `id` is captured by the closure. If the parent passes a new `id`, the component re-renders but the effect does not re-run, so `fetchData` is never called with the new `id`. The displayed data is stale — a classic missing-dependency bug. The ESLint `exhaustive-deps` rule would flag this.

Fix: `useEffect(() => { ... }, [id])`.

</details>

---

## 5. Async Effects

### Q19

```jsx
useEffect(async () => {
  const data = await fetch('/api').then(r => r.json());
  setData(data);
}, []);
```

#### ❓ What is the problem with this code?

<details>
<summary>✅ Answer</summary>

```txt
React will log a warning:

Warning: An effect function must not return anything besides a function, 
which is used for clean-up. You returned: [object Promise]
```

**Explanation:** `async` functions always return a Promise. `useEffect` expects the callback to return either `undefined` or a cleanup function. When it receives a Promise, it logs a warning and ignores it. No cleanup can be registered, so if the component unmounts before the fetch completes, the `.then` callback will still fire and attempt to call `setData` on an unmounted component — causing a potential state-update-on-unmounted-component warning and a possible memory leak.

</details>

---

### Q20

```jsx
function App({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    fetch(`/api/search?q=${query}`)
      .then(r => r.json())
      .then(data => setResults(data));
  }, [query]);

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}
```

#### ❓ If `query` changes rapidly (e.g., user is typing), what can go wrong?

<details>
<summary>✅ Answer</summary>

```txt
Race condition: the UI can display results from a stale (earlier) query.
```

**Explanation:** Each keystroke triggers a new fetch. Requests can resolve out of order. If the request for query `"re"` resolves after the request for `"rea"`, `setResults` is called with the results for `"re"`, displaying incorrect data. The fix is to use an `ignore` flag or `AbortController` in the cleanup.

</details>

---

### Q21

```jsx
function App({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let ignore = false;

    async function load() {
      const data = await fetchUser(userId);
      if (!ignore) setUser(data);
    }

    load();

    return () => { ignore = true; };
  }, [userId]);

  return <p>{user?.name}</p>;
}
```

#### ❓ What does the `ignore` flag prevent?

<details>
<summary>✅ Answer</summary>

```txt
It prevents a stale (out-of-order) response from overwriting the current data.
```

**Explanation:** When `userId` changes, the cleanup of the previous effect sets `ignore = true`. If the previous fetch resolves after the new one, `if (!ignore) setUser(data)` prevents the stale data from being applied. The component always shows data for the most recently requested `userId`. This is the ignore-flag pattern for preventing race conditions.

</details>

---

### Q22

```jsx
function App() {
  const [data, setData] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    fetch('/api/data', { signal: controller.signal })
      .then(r => r.json())
      .then(d => setData(d))
      .catch(err => console.log(err.name));

    return () => controller.abort();
  }, []);

  return <p>{data}</p>;
}
```

#### ❓ If the component unmounts immediately after mounting, what is logged?

<details>
<summary>✅ Answer</summary>

```txt
AbortError
```

**Explanation:** When the component unmounts, the cleanup fires and calls `controller.abort()`. This causes the in-flight fetch to reject with a `DOMException` of type `AbortError`. The `.catch` handler receives this error and logs `err.name`, which is `"AbortError"`. In production code, this should be handled: `if (err.name === 'AbortError') return;`.

</details>

---

## 6. Multiple Effects

### Q23

```jsx
function App() {
  const [a, setA] = useState(0);

  useEffect(() => {
    console.log('effect 1');
    return () => console.log('cleanup 1');
  }, [a]);

  useEffect(() => {
    console.log('effect 2');
    return () => console.log('cleanup 2');
  }, [a]);

  return <button onClick={() => setA(1)}>Click</button>;
}
```

#### ❓ What is logged on mount? What is logged after the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
// On mount:
effect 1
effect 2

// After click (a changes to 1):
cleanup 1
cleanup 2
effect 1
effect 2
```

**Explanation:** On mount both effects run in declaration order. When `a` changes, React runs all cleanups for effects whose deps changed (in order), then runs all new effects (in order). Cleanup always precedes re-setup, and both happen in declaration order.

</details>

---

### Q24

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  useEffect(() => {
    console.log('effect 1 - a:', a);
  }, [a]);

  useEffect(() => {
    console.log('effect 2 - b:', b);
  }, [b]);

  return (
    <button onClick={() => { setA(1); setB(1); }}>
      Update both
    </button>
  );
}
```

#### ❓ What is logged after the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
effect 1 - a: 1
effect 2 - b: 1
```

**Explanation:** React batches the two state updates from the same event handler into a single re-render (automatic batching in React 18). After the single re-render, both effects run in declaration order because both `a` and `b` changed.

</details>

---

### Q25

```jsx
function App() {
  useEffect(() => {
    console.log('effect A');
    return () => console.log('cleanup A');
  }, []);

  useEffect(() => {
    console.log('effect B');
    return () => console.log('cleanup B');
  }, []);

  return <div />;
}
```

#### ❓ What is logged when the component unmounts?

<details>
<summary>✅ Answer</summary>

```txt
// On mount:
effect A
effect B

// On unmount:
cleanup A
cleanup B
```

**Explanation:** On unmount, React runs all cleanup functions in the same order as the effects that registered them — declaration order. Both effects have `[]` so their cleanup only runs on unmount.

</details>

---

## 7. Edge Cases

### Q26

```jsx
function App() {
  const [show, setShow] = useState(false);

  useEffect(() => {
    if (show) {
      console.log('show is true');
    }
  }, [show]);

  return <button onClick={() => setShow(true)}>Show</button>;
}
```

#### ❓ Does the effect run on initial mount even though `show` is `false`?

<details>
<summary>✅ Answer</summary>

```txt
Yes — the effect runs on mount but prints nothing because the `if (show)` guard is false.
```

**Explanation:** `useEffect` with `[show]` always runs on initial mount, regardless of the value of `show`. The `if (show)` condition is evaluated inside the effect, so nothing is logged. After the button click, `show` becomes `true`, the effect re-runs, and `"show is true"` is logged.

</details>

---

### Q27

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('effect');
  }, [count]);

  setCount(0); // called during render

  return <p>{count}</p>;
}
```

#### ❓ What happens when this component renders?

<details>
<summary>✅ Answer</summary>

```txt
React enters an infinite render loop and throws a "Too many re-renders" error.
```

**Explanation:** Calling `setCount` directly during the render phase (not inside an event handler or useEffect) triggers a state update immediately after render, which triggers another render, which calls `setCount` again — infinitely. React detects this and throws. State updates must happen in event handlers, useEffect callbacks, or other side-effect contexts, never directly in the render function body.

</details>

---

### Q28

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('rendered with count:', count);
    setCount(count + 1);
  }, [count]);

  return <p>{count}</p>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

```txt
Infinite loop. Console fills with:
rendered with count: 0
rendered with count: 1
rendered with count: 2
...
```

**Explanation:** The effect runs after every render where `count` changed. It calls `setCount(count + 1)`, which changes `count`, which triggers a re-render, which fires the effect again. There is no termination condition, so this loops forever. In a browser, React's call stack eventually overflows.

</details>

---

### Q29

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    console.log('effect');
  }, [count]);

  return (
    <div>
      <button onClick={() => setCount(count)}>Set same value</button>
      <p>{count}</p>
    </div>
  );
}
```

#### ❓ Does the effect re-run when "Set same value" is clicked?

<details>
<summary>✅ Answer</summary>

```txt
No.
```

**Explanation:** When `setCount(count)` is called with the same value that `count` already holds, React performs a bailout optimization. If the state is the same (by `Object.is`), React does not re-render the component. Because there is no re-render, there is no new render for the effect to compare against, and the effect does not fire. This behavior is guaranteed for primitive state values.

</details>

---

### Q30

```jsx
function Parent() {
  const [x, setX] = useState(0);
  return <Child value={x} onChange={setX} />;
}

function Child({ value, onChange }) {
  useEffect(() => {
    onChange(value + 1);
  }, [value, onChange]);

  return <p>{value}</p>;
}
```

#### ❓ What happens when `Parent` renders for the first time?

<details>
<summary>✅ Answer</summary>

```txt
Infinite re-render loop.

value: 0 → effect fires → onChange(1) → value: 1 → effect fires → onChange(2) → ...
```

**Explanation:** On mount, `value = 0` and the effect fires calling `onChange(1)`. This updates Parent's state to `1`, causing Parent to re-render and pass `value = 1` to Child. Child re-renders, the effect fires again calling `onChange(2)`, and so on infinitely. There is no condition to stop the cycle. Effects that unconditionally call state setters based on their own deps always risk infinite loops.

</details>

---

## ✅ Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Timing | Q1 – Q4 | render before effect, useLayoutEffect order, no-dep vs empty-dep |
| Cleanup | Q5 – Q9 | unmount cleanup, cleanup before re-run, Object.is on same value, StrictMode double-mount, invalid return |
| Stale Closures | Q10 – Q14 | interval with captured state, setTimeout closure, functional update fix, useRef fix, event listener stale closure |
| Dependency Array | Q15 – Q18 | object reference instability, Object.is(NaN, NaN), function reference instability, missing dep causing stale data |
| Async Effects | Q19 – Q22 | async callback warning, race condition, ignore flag, AbortError |
| Multiple Effects | Q23 – Q25 | declaration order, cleanup order, automatic batching |
| Edge Cases | Q26 – Q30 | effect always runs on mount, setState during render, effect calling setState on its own dep, bailout on same value, infinite loop via child effect |
