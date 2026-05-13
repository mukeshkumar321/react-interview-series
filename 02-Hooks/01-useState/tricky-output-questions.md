## useState — Tricky Output Questions

> These questions test your understanding of async state updates, closures, batching, object/array immutability, and edge cases in `useState`. Each question reflects a real scenario you may encounter in a senior React interview.

---

## 1. Async Updates

### Q1

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ What does the button display after one click?

<details>
<summary>✅ Answer</summary>

```txt
1
```

**Explanation:** All three `setCount(count + 1)` calls read the same snapshot of `count = 0` from the current render. React batches them into a single re-render. Each call schedules `setCount(0 + 1)`, so the result is `1`, not `3`. To correctly increment three times, use the functional update form: `setCount(prev => prev + 1)` repeated three times would give `3`.

</details>

---

### Q2

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log('before:', count);
    setCount(count + 1);
    console.log('after:', count);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ What is logged when the button is clicked (initial count is 0)?

<details>
<summary>✅ Answer</summary>

```txt
before: 0
after: 0
```

**Explanation:** `setCount` does not synchronously update the `count` variable. State updates are asynchronous — they schedule a re-render. The `count` variable inside `handleClick` is a fixed snapshot from the current render (`0`). Even after calling `setCount(1)`, the local `count` variable still holds `0` for the rest of the current event handler execution.

</details>

---

### Q3

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  console.log('render');

  function handleClick() {
    setA(1);
    setB(2);
  }

  return <button onClick={handleClick}>Click</button>;
}
```

#### ❓ How many times is "render" logged after the button is clicked in React 18?

<details>
<summary>✅ Answer</summary>

```txt
render   ← initial render
render   ← single re-render after click (batched)
```

**Explanation:** React 18 introduced automatic batching. Both `setA(1)` and `setB(2)` inside the same event handler are batched into a single re-render. So `"render"` is logged only once after the click, not twice. In React 17, the same event handler was already batched, so behavior here is the same — but in timeouts/promises, React 18 batches while React 17 did not.

</details>

---

### Q4

```jsx
function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  console.log('render');

  function handleClick() {
    setTimeout(() => {
      setA(1);
      setB(2);
    }, 0);
  }

  return <button onClick={handleClick}>Click</button>;
}
```

#### ❓ How many times does "render" log after clicking in React 18 vs React 17?

<details>
<summary>✅ Answer</summary>

```txt
// React 18:
render   ← initial render
render   ← single re-render (automatic batching inside setTimeout)

// React 17:
render   ← initial render
render   ← re-render from setA(1)
render   ← re-render from setB(2)
```

**Explanation:** In React 17, state updates inside `setTimeout` were not batched — each `setState` triggered its own synchronous render, causing two re-renders. React 18 introduced automatic batching everywhere, including `setTimeout`, `Promise` callbacks, and native event handlers, so both updates are batched into a single re-render.

</details>

---

### Q5

```jsx
import { flushSync } from 'react-dom';

function App() {
  const [a, setA] = useState(0);
  const [b, setB] = useState(0);

  console.log('render');

  function handleClick() {
    flushSync(() => {
      setA(1);
    });
    flushSync(() => {
      setB(2);
    });
  }

  return <button onClick={handleClick}>Click</button>;
}
```

#### ❓ How many times does "render" log after clicking?

<details>
<summary>✅ Answer</summary>

```txt
render   ← initial render
render   ← immediate re-render from flushSync for setA
render   ← immediate re-render from flushSync for setB
```

**Explanation:** `flushSync` forces React to flush any pending state updates synchronously before returning. Each `flushSync` call triggers an immediate re-render. This opts out of batching entirely. After the first `flushSync`, `a` is `1` and the DOM is updated. After the second `flushSync`, `b` is `2` and the DOM is updated again. Three renders total.

</details>

---

## 2. Functional Updates

### Q6

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ What does the button display after one click?

<details>
<summary>✅ Answer</summary>

```txt
3
```

**Explanation:** Unlike direct value updates, functional updates `prev => prev + 1` are queued and processed sequentially. React applies them in order: `0 → 1 → 2 → 3`. Each updater function receives the result of the previous one as `prev`. This is the key difference from `setCount(count + 1)` repeated three times, which would only yield `1`.

</details>

---

### Q7

```jsx
function App() {
  const [count, setCount] = useState(10);

  function handleClick() {
    setCount(prev => prev * 2);
    setCount(prev => prev + 5);
    setCount(prev => prev - 3);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ What is the final value of count after one click?

<details>
<summary>✅ Answer</summary>

```txt
22
```

**Explanation:** React processes the functional updaters in declaration order, chaining the result of each as the input to the next:
1. `10 * 2 = 20`
2. `20 + 5 = 25`
3. `25 - 3 = 22`

Final value: `22`. Functional updates guarantee sequential processing using the most recent queued state, not the snapshot from the render.

</details>

---

### Q8

```jsx
function Toggle() {
  const [isOn, setIsOn] = useState(false);

  function handleClick() {
    setIsOn(!isOn);
    setIsOn(!isOn);
  }

  return <button onClick={handleClick}>{isOn ? 'ON' : 'OFF'}</button>;
}
```

#### ❓ What does the button show after clicking once? After clicking twice?

<details>
<summary>✅ Answer</summary>

```txt
After first click: OFF  (same as before)
After second click: OFF (still the same)
```

**Explanation:** Both `setIsOn(!isOn)` calls capture the same snapshot `isOn = false`. `!false = true`, so React schedules `setIsOn(true)` twice, resulting in `true` after batching — wait, that would show ON. Let me re-examine: both calls read `isOn = false`, compute `!false = true`, so state becomes `true`. Display shows `ON` after first click.

Actually the correct answer: After first click the button shows `ON`. Both calls set to `!false = true`. The second call also sets to `true`. React batches: final state `true`. The fix is `setIsOn(prev => !prev)` which would correctly toggle twice back to `false`.

```txt
After first click: ON
After second click: OFF (if clicking again when ON, same pattern: !true = false, !true = false → OFF)
```

**Explanation:** Both `setIsOn(!isOn)` calls read the same snapshot value. If `isOn = false`, both compute `true`. React batches them — result is `true` (ON). The double-toggle does not cancel itself because it's not using `prev`. To double-toggle back to the original value, use `setIsOn(prev => !prev)` twice, which would process `false → true → false`.

</details>

---

### Q9

```jsx
function App() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount(prev => {
      if (prev < 5) return prev + 1;
      return prev;
    });
  }

  return (
    <>
      <p>{count}</p>
      <button onClick={increment}>Increment</button>
    </>
  );
}
```

#### ❓ If the button is clicked 10 times, what does the counter show?

<details>
<summary>✅ Answer</summary>

```txt
5
```

**Explanation:** The functional updater includes a guard: if `prev >= 5`, it returns `prev` unchanged. When the same value is returned from an updater, React performs a bailout — no re-render occurs. The counter increments from 0 to 5, then every subsequent click returns `5`, which equals the current state, so React skips re-rendering after the 5th click.

</details>

---

### Q10

```jsx
function App() {
  const [value, setValue] = useState('hello');

  function handleClick() {
    setValue(prev => prev.toUpperCase());
    setValue(prev => prev + '!');
    setValue(prev => prev.split('').reverse().join(''));
  }

  return <button onClick={handleClick}>{value}</button>;
}
```

#### ❓ What is displayed after clicking the button once (starting value is "hello")?

<details>
<summary>✅ Answer</summary>

```txt
!OLLEH
```

**Explanation:** The three functional updaters chain:
1. `'hello'.toUpperCase()` → `'HELLO'`
2. `'HELLO' + '!'` → `'HELLO!'`
3. `'HELLO!'.split('').reverse().join('')` → `'!OLLEH'`

Each updater receives the result of the previous one as `prev`. The final state is `'!OLLEH'`.

</details>

---

## 3. Stale Closures

### Q11

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setTimeout(() => {
      setCount(count + 1);
      console.log('count in timeout:', count);
    }, 3000);
  }

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={handleClick}>Schedule +1 in 3s</button>
    </>
  );
}
```

#### ❓ "Schedule" is clicked when count is 0, then "Increment" is clicked 5 times before the 3 seconds elapse. What does the timeout log, and what is the final count value?

<details>
<summary>✅ Answer</summary>

```txt
count in timeout: 0
final count: 6
```

**Explanation:** The timeout closure captured `count = 0` at the time the "Schedule" button was clicked. When the timeout fires, it logs `0` (the stale snapshot) and calls `setCount(0 + 1) = setCount(1)`. Meanwhile, the five "Increment" clicks brought count to `5`. The timeout then sets count to `1`, but wait — the timeout sets to `count + 1` where `count` is the stale `0`, so it calls `setCount(1)`. But the state is already `5`. React sees a new value `1 ≠ 5` and re-renders with `1`.

Actually final count is `1` — the timeout overwrites the accumulated increments with a stale `setCount(0 + 1) = 1`. The fix: use `setCount(prev => prev + 1)` in the timeout.

```txt
count in timeout: 0
final count: 1
```

**Explanation:** The setTimeout callback closes over `count = 0`. It calls `setCount(0 + 1)`. After 5 increments, count is `5`. Then the stale timeout fires `setCount(1)` — overwriting the current `5` with `1`. This is the danger of stale closures with direct state updates.

</details>

---

### Q12

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      console.log(count);
    }, 1000);

    return () => clearInterval(id);
  }, []);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </>
  );
}
```

#### ❓ The button is clicked 3 times quickly. What does the interval log every second?

<details>
<summary>✅ Answer</summary>

```txt
0
0
0
0
... (always 0)
```

**Explanation:** The `useEffect` runs only once on mount (empty deps array). The `setInterval` callback closes over `count = 0` from that render. Even though clicking the button updates `count` in the component's state and re-renders the UI, the interval callback is a closure that was created with the original `count = 0` and never re-created. It always logs `0`. Fix: use a ref synchronized with state, or add `count` to the deps (which would restart the interval on each change).

</details>

---

### Q13

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const handler = () => {
      console.log('clicked, count is:', count);
    };

    window.addEventListener('click', handler);
    return () => window.removeEventListener('click', handler);
  }, []);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </>
  );
}
```

#### ❓ The counter is incremented to 3. Then the window is clicked. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
clicked, count is: 0
```

**Explanation:** The window event listener is registered once on mount with `count = 0`. The closure captures `count = 0` permanently. Clicking the Increment button updates React state and re-renders the `<p>` with the new count, but the event listener closure is never replaced (effect has empty deps). So the listener always sees `count = 0`. To fix: add `count` to the deps array (or use a ref).

</details>

---

### Q14

```jsx
function App() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;
  }, [count]);

  useEffect(() => {
    const handler = () => {
      console.log('count via ref:', countRef.current);
    };

    window.addEventListener('keydown', handler);
    return () => window.removeEventListener('keydown', handler);
  }, []);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </>
  );
}
```

#### ❓ The button is clicked 4 times, then any key is pressed. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
count via ref: 4
```

**Explanation:** The `countRef` acts as a bridge between stale closures and fresh state. Every time `count` changes, the first `useEffect` updates `countRef.current` to the latest value. The keydown handler closes over `countRef` (the object), not `countRef.current` (the value). When the handler runs, it reads `.current` at that moment — which is the most recently set value, `4`. This is the canonical pattern for reading latest state in stale closures.

</details>

---

### Q15

```jsx
function App() {
  const [name, setName] = useState('Alice');

  const greet = useCallback(() => {
    console.log('Hello,', name);
  }, []); // empty deps — missing name

  return (
    <>
      <button onClick={() => setName('Bob')}>Change Name</button>
      <button onClick={greet}>Greet</button>
    </>
  );
}
```

#### ❓ "Change Name" is clicked first, then "Greet". What is logged?

<details>
<summary>✅ Answer</summary>

```txt
Hello, Alice
```

**Explanation:** `useCallback` with an empty dependency array memoizes the `greet` function from the first render. It closes over `name = 'Alice'` and is never recreated. Even after `name` changes to `'Bob'`, `greet` still holds the stale closure. The ESLint exhaustive-deps rule would flag this: `name` should be in the dependency array. Fix: `useCallback(() => { console.log('Hello,', name); }, [name])`.

</details>

---

## 4. Object / Array State

### Q16

```jsx
function App() {
  const [user, setUser] = useState({ name: 'Alice', age: 25 });

  function handleClick() {
    user.name = 'Bob';
    setUser(user);
  }

  return (
    <>
      <p>{user.name}</p>
      <button onClick={handleClick}>Change Name</button>
    </>
  );
}
```

#### ❓ Does clicking the button update the displayed name?

<details>
<summary>✅ Answer</summary>

```txt
May or may not update — behavior is unreliable. In most cases the UI does NOT update.
```

**Explanation:** `user.name = 'Bob'` mutates the state object directly. Then `setUser(user)` passes the same object reference. React compares old state vs new state using `Object.is(oldUser, newUser)`. Since it is the same object reference, `Object.is` returns `true`, and React bails out — no re-render occurs. Even though you mutated the internal values, React sees no state change. Always create a new object: `setUser(prev => ({ ...prev, name: 'Bob' }))`.

</details>

---

### Q17

```jsx
function App() {
  const [user, setUser] = useState({ name: 'Alice', age: 25 });

  function updateAge() {
    setUser(prev => ({ ...prev, age: 26 }));
  }

  function updateName() {
    setUser(prev => ({ ...prev, name: 'Bob' }));
  }

  return (
    <>
      <p>{user.name}, {user.age}</p>
      <button onClick={updateAge}>+1 Age</button>
      <button onClick={updateName}>Change Name</button>
    </>
  );
}
```

#### ❓ After clicking "+1 Age" then "Change Name", what is displayed?

<details>
<summary>✅ Answer</summary>

```txt
Bob, 26
```

**Explanation:** Both updates correctly spread the previous state, so no fields are lost. After "+1 Age": `{ name: 'Alice', age: 26 }`. After "Change Name": `{ name: 'Bob', age: 26 }`. The spread operator `{ ...prev, ... }` preserves all existing fields and only overrides the specified one. This is the correct immutable update pattern for object state.

</details>

---

### Q18

```jsx
function App() {
  const [profile, setProfile] = useState({
    name: 'Alice',
    address: { city: 'Mumbai', pin: '400001' }
  });

  function updateCity() {
    setProfile(prev => ({
      ...prev,
      address: { city: 'Delhi' }  // forgot to spread address
    }));
  }

  return (
    <>
      <p>{profile.address.city}</p>
      <p>{profile.address.pin}</p>
      <button onClick={updateCity}>Move to Delhi</button>
    </>
  );
}
```

#### ❓ After clicking "Move to Delhi", what do the two `<p>` elements display?

<details>
<summary>✅ Answer</summary>

```txt
Delhi
undefined
```

**Explanation:** The `address` is replaced entirely with `{ city: 'Delhi' }` — the `pin` field is lost because the inner spread was omitted. This is a common nested object mutation mistake. The outer `...prev` preserves `name`, but `address` is replaced with a brand new object that only has `city`. The correct fix: `address: { ...prev.address, city: 'Delhi' }` to preserve `pin`.

</details>

---

### Q19

```jsx
function App() {
  const [items, setItems] = useState([1, 2, 3]);

  function addItem() {
    items.push(4);
    setItems(items);
  }

  return (
    <>
      <ul>{items.map(i => <li key={i}>{i}</li>)}</ul>
      <button onClick={addItem}>Add 4</button>
    </>
  );
}
```

#### ❓ Does clicking "Add 4" update the list?

<details>
<summary>✅ Answer</summary>

```txt
No update in most cases — the list stays at [1, 2, 3] in the UI.
```

**Explanation:** `items.push(4)` mutates the existing array in place. `setItems(items)` then passes the same array reference. React's `Object.is(oldItems, newItems)` compares references — they are the same array, so React bails out and skips re-rendering. Even though `items` was mutated to contain 4 elements, the displayed list does not update. Always create a new array: `setItems(prev => [...prev, 4])`.

</details>

---

### Q20

```jsx
function App() {
  const [todos, setTodos] = useState([
    { id: 1, text: 'Buy milk', done: false },
    { id: 2, text: 'Read book', done: false },
  ]);

  function toggleTodo(id) {
    setTodos(prev =>
      prev.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      )
    );
  }

  return (
    <ul>
      {todos.map(todo => (
        <li
          key={todo.id}
          onClick={() => toggleTodo(todo.id)}
          style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
        >
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

#### ❓ After clicking "Buy milk", what is the state of `todos`?

<details>
<summary>✅ Answer</summary>

```txt
[
  { id: 1, text: 'Buy milk', done: true },
  { id: 2, text: 'Read book', done: false }
]
```

**Explanation:** `map` returns a new array. For `id === 1`, `{ ...todo, done: !todo.done }` creates a new object with `done: true`. For `id !== 1`, the original todo object is returned unchanged (same reference). The result is a new array with a new object at position 0 and the same object at position 1. This is the correct immutable update pattern for arrays of objects.

</details>

---

## 5. Initial State

### Q21

```jsx
function App() {
  const [value, setValue] = useState(() => {
    console.log('initializer');
    return 42;
  });

  console.log('render');

  return (
    <>
      <p>{value}</p>
      <button onClick={() => setValue(v => v + 1)}>+</button>
    </>
  );
}
```

#### ❓ What is logged on the initial render and after clicking the button 3 times?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
initializer
render

// After each button click (3 times):
render
render
render
```

**Explanation:** The lazy initializer function is called only once — on the first render — to compute the initial state. Subsequent renders ignore the initializer entirely. The component body ("render") executes on every render. After 3 clicks, "render" is logged 3 more times, but "initializer" is never logged again. This is the key benefit of lazy initialization: avoiding expensive computations on every render.

</details>

---

### Q22

```jsx
function App() {
  const [data, setData] = useState(JSON.parse(localStorage.getItem('data') || '[]'));

  console.log('render');

  return <p>{data.length} items</p>;
}
```

#### ❓ What is the problem with this code?

<details>
<summary>✅ Answer</summary>

```txt
JSON.parse(localStorage.getItem('data') || '[]') runs on every render, not just the first.
```

**Explanation:** When you pass a value directly to `useState`, that expression is evaluated on every render — even though `useState` only uses it on the first render. `JSON.parse` and `localStorage.getItem` are called on every component re-render, wasting CPU. The fix is to use lazy initialization: `useState(() => JSON.parse(localStorage.getItem('data') || '[]'))`. Passing a function makes React call it only on the first render.

</details>

---

### Q23

```jsx
function SearchInput({ defaultQuery }) {
  const [query, setQuery] = useState(defaultQuery);

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
    />
  );
}

function App() {
  const [search, setSearch] = useState('react');

  return (
    <>
      <SearchInput defaultQuery={search} />
      <button onClick={() => setSearch('hooks')}>Change Default</button>
    </>
  );
}
```

#### ❓ After clicking "Change Default", does the input update to show "hooks"?

<details>
<summary>✅ Answer</summary>

```txt
No — the input still shows "react".
```

**Explanation:** `useState(defaultQuery)` uses `defaultQuery` only on the first render to initialize state. On subsequent renders, `useState` ignores its argument and returns the stored state. Changing `search` to `'hooks'` causes `SearchInput` to re-render with the new `defaultQuery` prop, but `query` state remains `'react'`. This is the state initialization from props anti-pattern. To sync with parent, either use the prop directly (controlled) or use a `key` to remount the component.

</details>

---

### Q24

```jsx
function App() {
  const [count, setCount] = useState(() => {
    console.log('lazy init');
    return 0;
  });

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ In React 18 Strict Mode (development), how many times is "lazy init" logged?

<details>
<summary>✅ Answer</summary>

```txt
2
```

**Explanation:** React 18 Strict Mode double-invokes the lazy initializer function to verify it is pure (no side effects). The initializer is called twice in development to check that calling it twice produces the same result. Only the first result is used. In production, the initializer is called exactly once. This is also true for component render functions, reducer functions, and functions passed to `useMemo`.

</details>

---

### Q25

```jsx
let initCount = 0;

function App() {
  const [value] = useState(() => {
    initCount++;
    return 10;
  });

  return <p>value: {value}, initCount: {initCount}</p>;
}
```

#### ❓ What is displayed (in production)? What about in Strict Mode development?

<details>
<summary>✅ Answer</summary>

```txt
// Production:
value: 10, initCount: 1

// Strict Mode development:
value: 10, initCount: 2
```

**Explanation:** In production, the initializer runs exactly once. In Strict Mode development, React double-invokes the initializer to check for purity. `initCount` is a module-level variable that gets incremented each time the initializer runs. Since `initCount` is not state, the UI doesn't automatically re-render when it changes, but the final render shows the accumulated value. Production: `1`. Strict Mode: `2`.

</details>

---

## 6. Edge Cases

### Q26

```jsx
function App() {
  const [count, setCount] = useState(0);

  console.log('render');

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(0)}>Set to 0</button>
    </>
  );
}
```

#### ❓ After the initial render, "Set to 0" is clicked 5 times. How many times does "render" log total?

<details>
<summary>✅ Answer</summary>

```txt
1 (only the initial render)

Or possibly 2 (initial render + one extra due to React's bailout logic).
```

**Explanation:** When `setCount(0)` is called with the same value that `count` already holds (`0`), React uses `Object.is(0, 0) = true` to detect no change and bails out — skipping the re-render. So clicking "Set to 0" when count is already `0` produces no additional renders. Note: React may still run the component function once to confirm the bailout, but no DOM update happens. After the first bailout confirmation, subsequent calls with the same value skip even that single run.

</details>

---

### Q27

```jsx
function App() {
  const [count, setCount] = useState(0);

  if (count === 0) {
    setCount(1); // called during render
  }

  return <p>{count}</p>;
}
```

#### ❓ What happens when this renders?

<details>
<summary>✅ Answer</summary>

```txt
React throws: "Too many re-renders. React limits the number of renders to prevent an infinite loop."
```

**Explanation:** Calling `setCount` unconditionally during render (outside of an event handler or effect) triggers an infinite loop. The component renders with `count = 0`, `setCount(1)` schedules a re-render, count becomes `1`, the condition `count === 0` is false so `setCount` is not called again. Wait — actually this would stop after count reaches `1`. Let me re-examine: on first render, count is `0`, `setCount(1)` is called. React immediately re-renders with `count = 1`. Now `count === 0` is false, so no more setState. React allows calling setState during render only in specific guarded patterns, but any cyclic pattern causes "Too many re-renders".

Actually if count changes to 1, the loop stops. But React still detects the pattern and may warn. The real infinite loop version would be `setCount(count + 1)` without a guard.

```txt
Displays: 1
(React allows setState during render as a derived state update, but only if the condition terminates)
```

**Explanation:** This specific code would display `1`. React allows `setState` during render for derived state patterns, but only if the update terminates. First render: `count = 0` → `setCount(1)` → re-render with `count = 1` → condition false → no more setState → displays `1`. However, calling setState unconditionally during render (e.g., without a guard) causes infinite loops and is generally an anti-pattern.

</details>

---

### Q28

```jsx
function App() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <>
      <p>{count}</p>
      <button onClick={handleClick}>+1</button>
    </>
  );
}
```

#### ❓ `Object.is` is used internally by React to compare state. Which of these `setCount` calls will cause a re-render?
```jsx
setCount(0);    // current count is 0
setCount(NaN);  // current count is NaN
setCount(+0);   // current count is -0
setCount(-0);   // current count is +0
```

<details>
<summary>✅ Answer</summary>

```txt
setCount(0)  when count is 0  → NO re-render  (Object.is(0, 0) = true)
setCount(NaN) when count is NaN → NO re-render (Object.is(NaN, NaN) = true)
setCount(+0)  when count is -0  → RE-RENDER    (Object.is(+0, -0) = false)
setCount(-0)  when count is +0  → RE-RENDER    (Object.is(-0, +0) = false)
```

**Explanation:** React uses `Object.is` for state comparison, which differs from `===` in two cases:
1. `Object.is(NaN, NaN)` is `true` (unlike `NaN !== NaN`)
2. `Object.is(+0, -0)` is `false` (unlike `+0 === -0`)

So `NaN` to `NaN` is a bailout (no re-render), but `+0` to `-0` (or vice versa) triggers a re-render.

</details>

---

### Q29

```jsx
function App() {
  const [items] = useState(['a', 'b', 'c']);
  const count = items.length; // derived from state

  return (
    <>
      <p>Count: {count}</p>
      <p>First: {items[0]}</p>
    </>
  );
}
```

#### ❓ Is storing `count` as a separate state variable a problem here? What would happen if someone did `const [count, setCount] = useState(items.length)`?

<details>
<summary>✅ Answer</summary>

```txt
No problem here — count is correctly derived during render, not stored in state.

If count were stored as separate state: it would become stale when items changes.
```

**Explanation:** Computing `count` directly from `items.length` during render is the correct derived state pattern. It is always in sync with `items`. If instead you wrote `const [count, setCount] = useState(items.length)`, you would have duplicate state that can go out of sync — if `items` updates, you also need to remember to call `setCount`. React's model encourages computing derived values from the single source of truth rather than storing redundant derived state.

</details>

---

### Q30

```jsx
function EditForm({ userId, serverData }) {
  const [form, setForm] = useState(serverData);

  return (
    <input
      value={form.name}
      onChange={e => setForm(prev => ({ ...prev, name: e.target.value }))}
    />
  );
}

function App() {
  const [userId, setUserId] = useState(1);
  const data = { id: userId, name: userId === 1 ? 'Alice' : 'Bob' };

  return (
    <>
      <EditForm key={userId} userId={userId} serverData={data} />
      <button onClick={() => setUserId(2)}>Switch to User 2</button>
    </>
  );
}
```

#### ❓ After switching to User 2, does the form show "Bob" or does it retain the edits from User 1?

<details>
<summary>✅ Answer</summary>

```txt
The form shows "Bob" — all edits from User 1 are discarded.
```

**Explanation:** The `key={userId}` prop is the critical detail. When `userId` changes from `1` to `2`, React sees a different `key` on `<EditForm>` and treats it as a completely different component instance. It unmounts the old `EditForm` (discarding all its state, including any edits) and mounts a brand new `EditForm` with `serverData` for User 2. This is the "key reset pattern" — the intentional way to reset component state when an identifier changes.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Async Updates | Q1 – Q5 | snapshot-based setState, batching, flushSync, React 18 batching in setTimeout |
| Functional Updates | Q6 – Q10 | sequential processing, chaining updaters, toggle pattern, conditional guard, string transformations |
| Stale Closures | Q11 – Q15 | setTimeout stale snapshot, setInterval closure, event listener closure, useRef fix, useCallback stale dep |
| Object / Array State | Q16 – Q20 | direct mutation bailout, spread pattern, nested object missing spread, array push mutation, immutable array update |
| Initial State | Q21 – Q25 | lazy init runs once, expensive init on every render, prop init anti-pattern, StrictMode double-invoke, production vs dev |
| Edge Cases | Q26 – Q30 | same-value bailout, setState during render, Object.is comparison, derived state, key reset pattern |
