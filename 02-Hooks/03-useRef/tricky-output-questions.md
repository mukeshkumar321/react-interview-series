## useRef — Tricky Output Questions

> These questions test your understanding of DOM access timing, mutable values, stale closure fixes, forwardRef, and edge cases in `useRef`. Each question reflects a real scenario from senior React interviews.

---

## 1. DOM Access

### Q1

```jsx
function App() {
  const divRef = useRef(null);

  console.log('during render:', divRef.current);

  useEffect(() => {
    console.log('in effect:', divRef.current);
  }, []);

  return <div ref={divRef}>Hello</div>;
}
```

#### ❓ What is logged during render and inside the effect?

<details>
<summary>✅ Answer</summary>

```txt
during render: null
in effect: <div>Hello</div>  (the actual DOM node)
```

**Explanation:** DOM refs are populated by React after the component's JSX has been committed to the DOM. During the render phase, `divRef.current` is still `null` — the DOM element does not exist yet. After the DOM commit, React sets `ref.current` to the DOM node. `useEffect` runs after the DOM commit, so `ref.current` is the actual DOM element there. Never read DOM refs during the render phase.

</details>

---

### Q2

```jsx
function App() {
  const [show, setShow] = useState(true);
  const inputRef = useRef(null);

  const focusInput = () => {
    console.log('ref.current:', inputRef.current);
    inputRef.current?.focus();
  };

  return (
    <>
      <button onClick={() => setShow(s => !s)}>Toggle</button>
      {show && <input ref={inputRef} />}
      <button onClick={focusInput}>Focus</button>
    </>
  );
}
```

#### ❓ "Toggle" is clicked (hiding the input), then "Focus" is clicked. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
ref.current: null
```

**Explanation:** When the input is removed from the DOM (because `show` becomes `false`), React automatically resets `ref.current` to `null`. The optional chaining `?.focus()` prevents a TypeError crash. After the element is removed from the DOM, the ref no longer points to any element. This is the automatic ref cleanup React performs on unmount or conditional removal.

</details>

---

### Q3

```jsx
function App() {
  const buttonRef = useRef(null);

  const handleClick = () => {
    console.log('button width:', buttonRef.current.offsetWidth);
  };

  return (
    <button ref={buttonRef} onClick={handleClick}>
      Measure Me
    </button>
  );
}
```

#### ❓ Is it safe to access `buttonRef.current.offsetWidth` inside the click handler?

<details>
<summary>✅ Answer</summary>

```txt
Yes — it is safe and will log the button's pixel width.
```

**Explanation:** Event handlers fire after the component has mounted and the DOM is fully committed. At the time a click event fires, `buttonRef.current` is always the button DOM element (since the button is in the DOM for the user to click it). Reading DOM properties like `offsetWidth` inside event handlers is completely safe. The danger is reading refs during the render phase, not in event handlers or effects.

</details>

---

### Q4

```jsx
function SearchBar() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus();
  }, []);

  return <input ref={inputRef} placeholder="Search..." />;
}
```

#### ❓ What happens when this component mounts?

<details>
<summary>✅ Answer</summary>

```txt
The input receives focus automatically on mount.
```

**Explanation:** `useEffect` with `[]` runs once after the first render and DOM commit. At that point, `inputRef.current` is the `<input>` DOM element. Calling `.focus()` moves keyboard focus to the input. This is the standard pattern for auto-focusing an element on mount. In Strict Mode development, the effect runs twice (mount → cleanup → remount), but the focus behavior is the same.

</details>

---

### Q5

```jsx
function Gallery({ images }) {
  const refsRef = useRef([]);

  const scrollTo = (index) => {
    refsRef.current[index]?.scrollIntoView({ behavior: 'smooth' });
  };

  return (
    <div>
      {images.map((img, i) => (
        <img
          key={img.id}
          src={img.src}
          ref={el => (refsRef.current[i] = el)}
        />
      ))}
      <button onClick={() => scrollTo(2)}>Scroll to #3</button>
    </div>
  );
}
```

#### ❓ What type of ref technique is being used on each `<img>`? When is `refsRef.current[i]` set to `null`?

<details>
<summary>✅ Answer</summary>

```txt
A callback ref (ref callback) is used. refsRef.current[i] is set to null when
the image element is removed from the DOM (unmounted or image array shrinks).
```

**Explanation:** Instead of passing a ref object, a function `el => (refsRef.current[i] = el)` is passed to the `ref` prop. React calls this function with the DOM element on mount and with `null` on unmount. This allows storing multiple refs in an array. When an image is removed, React calls the callback with `null`, setting `refsRef.current[i] = null`. This is the callback ref pattern for dynamic lists of elements.

</details>

---

## 2. Mutable Values

### Q6

```jsx
function App() {
  const countRef = useRef(0);

  const increment = () => {
    countRef.current += 1;
    console.log('countRef.current:', countRef.current);
  };

  return (
    <>
      <p>Display: {countRef.current}</p>
      <button onClick={increment}>+</button>
    </>
  );
}
```

#### ❓ After clicking "+" 3 times, what does the `<p>` show? What does the console log?

<details>
<summary>✅ Answer</summary>

```txt
// Console (each click):
countRef.current: 1
countRef.current: 2
countRef.current: 3

// The <p> always shows: Display: 0
```

**Explanation:** Mutating `ref.current` does NOT trigger a re-render. The console logs accurately reflect the current ref value (1, 2, 3). However, the `<p>` element reads `countRef.current` during the render phase. Since no re-render occurs after the mutation, the JSX is never re-evaluated and the displayed value stays at `0` (the initial value from the first render). Use `useState` if you need to display the value in the UI.

</details>

---

### Q7

```jsx
function App() {
  const [count, setCount] = useState(0);
  const renderCount = useRef(0);
  renderCount.current += 1;

  return (
    <>
      <p>Count: {count}</p>
      <p>Renders: {renderCount.current}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </>
  );
}
```

#### ❓ After clicking "+" 3 times, what do both `<p>` elements show?

<details>
<summary>✅ Answer</summary>

```txt
Count: 3
Renders: 4
```

**Explanation:** `renderCount.current` is incremented every time the component renders (including the initial render). Initial render: `renderCount.current` becomes `1`. Each click triggers a re-render, incrementing `renderCount.current` by 1 each time. After 3 clicks: `1 (initial) + 3 (re-renders) = 4`. Note: in Strict Mode development, the initial render is doubled, so it would show `5` (2 initial + 3 clicks). In production without StrictMode: `4`.

</details>

---

### Q8

```jsx
function Stopwatch() {
  const [time, setTime] = useState(0);
  const intervalRef = useRef(null);

  const start = () => {
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

#### ❓ Why is `intervalRef` a `useRef` and not `useState`? What would happen if it were `useState`?

<details>
<summary>✅ Answer</summary>

```txt
useRef is correct because storing the interval ID does not need to trigger a re-render.

If useState were used: setIntervalId(id) would trigger an unnecessary re-render
each time start() is called.
```

**Explanation:** The interval ID is never displayed in the UI — it is an implementation detail needed to cancel the interval. Using `useState` for it would cause an extra re-render every time `start()` is called (to store the ID) and every time `stop()` is called (to reset it). `useRef` stores the ID without any rendering overhead. This is the canonical pattern: use `useRef` for values that don't belong in the UI.

</details>

---

### Q9

```jsx
function App() {
  const [count, setCount] = useState(0);
  const prevCountRef = useRef(undefined);

  useEffect(() => {
    prevCountRef.current = count;
  });

  const prevCount = prevCountRef.current;

  return (
    <>
      <p>Current: {count}</p>
      <p>Previous: {prevCount ?? 'none'}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </>
  );
}
```

#### ❓ What is displayed initially, after the first click, and after the second click?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
Current: 0
Previous: none

// After first click (count = 1):
Current: 1
Previous: 0

// After second click (count = 2):
Current: 2
Previous: 1
```

**Explanation:** `useEffect` with no dependency array runs after every render. After each render, `prevCountRef.current` is updated to the current `count`. During the next render, `prevCountRef.current` still holds the previous render's value (before the effect updates it). This "one render behind" behavior is the usePrevious pattern. The effect runs after render, so during render N, the ref still contains render N-1's value.

</details>

---

### Q10

```jsx
function App() {
  const ref = useRef('initial');

  console.log('render, ref:', ref.current);

  ref.current = 'updated during render';

  return (
    <>
      <p>{ref.current}</p>
      <button onClick={() => ref.current = 'clicked'}>Click</button>
    </>
  );
}
```

#### ❓ After the initial render and then clicking the button, what does the `<p>` show each time?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render: <p> shows "updated during render"
// After clicking: <p> still shows "updated during render" (no re-render)
```

**Explanation:** During the initial render, `ref.current` is mutated to `'updated during render'` synchronously in the render body. The JSX then reads the updated value and displays it. After clicking, `ref.current` becomes `'clicked'`, but this mutation does NOT trigger a re-render. So the `<p>` continues displaying `'updated during render'` until something else causes a re-render. Mutating refs during render is generally discouraged and can lead to confusing behavior.

</details>

---

## 3. Stale Closure Fix

### Q11

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setTimeout(() => {
      console.log('timeout count:', count);
    }, 5000);
    return () => clearTimeout(id);
  }, []);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </>
  );
}
```

#### ❓ The button is clicked 5 times in the first 5 seconds. What does the timeout log?

<details>
<summary>✅ Answer</summary>

```txt
timeout count: 0
```

**Explanation:** The `useEffect` runs once on mount. The setTimeout callback closes over `count = 0` from that initial render. No matter how many times the button is clicked, the closed-over `count` is frozen at `0`. The component UI correctly shows `5`, but the timeout logs the stale `0`. This is the classic stale closure in async callbacks problem.

</details>

---

### Q12

```jsx
function App() {
  const [count, setCount] = useState(0);
  const countRef = useRef(0);

  useEffect(() => {
    countRef.current = count;
  }, [count]);

  useEffect(() => {
    const id = setTimeout(() => {
      console.log('timeout count:', countRef.current);
    }, 5000);
    return () => clearTimeout(id);
  }, []);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </>
  );
}
```

#### ❓ The button is clicked 5 times in the first 5 seconds. What does the timeout log?

<details>
<summary>✅ Answer</summary>

```txt
timeout count: 5
```

**Explanation:** The key difference from Q11 is the ref bridge. Every time `count` changes, the first `useEffect` updates `countRef.current` to the latest value. The timeout callback closes over `countRef` (the stable container object), not `countRef.current` (the value). When the timeout fires, it reads `.current` at that exact moment — which is the latest value `5`. The ref object reference is stable; reading `.current` at runtime bypasses the stale closure.

</details>

---

### Q13

```jsx
function App() {
  const [message, setMessage] = useState('hello');
  const messageRef = useRef(message);

  useEffect(() => {
    messageRef.current = message;
  }, [message]);

  const logLater = () => {
    setTimeout(() => {
      console.log('message:', messageRef.current);
    }, 2000);
  };

  return (
    <>
      <input value={message} onChange={e => setMessage(e.target.value)} />
      <button onClick={logLater}>Log in 2s</button>
    </>
  );
}
```

#### ❓ "Log in 2s" is clicked, then the input is changed to "world" before 2 seconds pass. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
message: world
```

**Explanation:** The ref is synchronized with state via `useEffect`. When `message` changes to `'world'`, `messageRef.current` is updated to `'world'`. The timeout that was scheduled reads `messageRef.current` at fire time (2 seconds later), finding the latest value `'world'`. This is the ref-as-bridge-for-stale-closures pattern: the closure captures the stable ref object, reads `.current` lazily at execution time.

</details>

---

### Q14

```jsx
function App() {
  const [value, setValue] = useState(0);

  const logValue = useCallback(() => {
    console.log('value:', value);
  }, [value]); // correctly listed as dep

  useEffect(() => {
    const id = setTimeout(logValue, 3000);
    return () => clearTimeout(id);
  }, [logValue]);

  return (
    <>
      <p>{value}</p>
      <button onClick={() => setValue(v => v + 1)}>+</button>
    </>
  );
}
```

#### ❓ The button is clicked 3 times. What does the timeout log?

<details>
<summary>✅ Answer</summary>

```txt
value: 3
```

**Explanation:** Each click changes `value`, which causes `logValue` to be recreated (because `value` is in its deps). A new `logValue` causes the effect to re-run (because `logValue` is in its deps), which cancels the old timeout and schedules a new one. After 3 clicks, `value = 3` and a new 3-second timeout is set. No previous timeout fires because they were all cancelled by cleanup. After 3 seconds of inactivity, the current `logValue` (which closes over `value = 3`) fires.

</details>

---

### Q15

```jsx
function App() {
  const [items, setItems] = useState([1, 2, 3]);
  const itemsRef = useRef(items);
  itemsRef.current = items; // update ref on every render

  const logItems = useCallback(() => {
    console.log('items:', itemsRef.current);
  }, []); // stable — never recreated

  return (
    <>
      <button onClick={() => setItems(prev => [...prev, prev.length + 1])}>
        Add
      </button>
      <button onClick={logItems}>Log Items</button>
    </>
  );
}
```

#### ❓ "Add" is clicked 3 times, then "Log Items" is clicked. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
items: [1, 2, 3, 4, 5, 6]
```

**Explanation:** `itemsRef.current = items` is assigned during every render — this is an inline ref update pattern that keeps the ref always in sync with the latest state. `logItems` is memoized with `[]` (never recreated) but reads `itemsRef.current` at call time. After 3 "Add" clicks, items is `[1, 2, 3, 4, 5, 6]` and the ref holds the latest array. Clicking "Log Items" logs `[1, 2, 3, 4, 5, 6]`. This avoids recreating callbacks while still reading fresh values.

</details>

---

## 4. forwardRef

### Q16

```jsx
const FancyInput = React.forwardRef((props, ref) => (
  <input ref={ref} className="fancy" {...props} />
));

function App() {
  const inputRef = useRef(null);

  const focusInput = () => {
    console.log('ref:', inputRef.current);
    inputRef.current?.focus();
  };

  return (
    <>
      <FancyInput ref={inputRef} placeholder="Type here" />
      <button onClick={focusInput}>Focus</button>
    </>
  );
}
```

#### ❓ After clicking "Focus", what is logged?

<details>
<summary>✅ Answer</summary>

```txt
ref: <input class="fancy" placeholder="Type here">  (the actual DOM input element)
```

**Explanation:** `React.forwardRef` passes the parent's `ref` as the second argument to the component's render function. The component forwards it to the inner `<input>` via `ref={ref}`. So `inputRef.current` in the parent points directly to the underlying DOM `<input>` element, not to the `FancyInput` component itself. The `.focus()` call works correctly.

</details>

---

### Q17

```jsx
function FancyInput(props) {
  return <input className="fancy" {...props} />;
}

function App() {
  const inputRef = useRef(null);

  useEffect(() => {
    console.log('ref.current:', inputRef.current);
  }, []);

  return <FancyInput ref={inputRef} />;
}
```

#### ❓ What is logged in the effect? Is there a warning?

<details>
<summary>✅ Answer</summary>

```txt
ref.current: null

Warning: Function components cannot be given refs. Attempts to access this ref will fail.
Did you mean to use React.forwardRef()?
```

**Explanation:** Passing a `ref` to a plain functional component (without `forwardRef`) does not work. React intercepts the `ref` prop before passing props to the component function. The `ref` is never forwarded to the inner `<input>`. `inputRef.current` remains `null`. React logs a warning in development. The fix is to wrap `FancyInput` with `React.forwardRef`.

</details>

---

### Q18

```jsx
const SmartInput = React.forwardRef((props, ref) => {
  const localRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() { localRef.current.focus(); },
    clear() { localRef.current.value = ''; },
  }));

  return <input ref={localRef} {...props} />;
});

function App() {
  const inputRef = useRef(null);

  useEffect(() => {
    console.log(Object.keys(inputRef.current));
    inputRef.current.focus();
    console.log(inputRef.current.style); // does this work?
  }, []);

  return <SmartInput ref={inputRef} />;
}
```

#### ❓ What does `Object.keys(inputRef.current)` log? Does `inputRef.current.style` work?

<details>
<summary>✅ Answer</summary>

```txt
Object.keys: ['focus', 'clear']
inputRef.current.style: undefined
```

**Explanation:** `useImperativeHandle` replaces the value that `ref.current` points to with a custom object. Instead of the raw DOM node, `inputRef.current` is `{ focus: fn, clear: fn }`. Only those two methods are exposed. `inputRef.current.style` is `undefined` because the raw DOM node is not exposed. This is the encapsulation benefit of `useImperativeHandle` — the parent can only use the API you explicitly define.

</details>

---

### Q19

```jsx
const Child = React.forwardRef(function Child(props, ref) {
  console.log('props.ref:', props.ref);
  console.log('ref arg:', ref);
  return <div ref={ref}>Child</div>;
});

function App() {
  const divRef = useRef(null);
  return <Child ref={divRef} data="test" />;
}
```

#### ❓ What does `props.ref` and `ref` (the second argument) log?

<details>
<summary>✅ Answer</summary>

```txt
props.ref: undefined
ref arg: { current: null }  (the ref object passed from parent)
```

**Explanation:** `ref` is not included in the `props` object — React intercepts it and passes it as the explicit second argument to the `forwardRef` render function. `props.ref` is `undefined` because `ref` never reaches `props`. The actual ref object `{ current: null }` arrives via the second argument `ref`. After the component mounts and React commits the DOM, `ref.current` will be the `<div>` element.

</details>

---

### Q20

```jsx
const Modal = React.forwardRef(({ children }, ref) => (
  <div className="modal" ref={ref}>
    {children}
  </div>
));

Modal.displayName = 'Modal';

function App() {
  const modalRef = useRef(null);

  const closeModal = () => {
    modalRef.current?.classList.remove('visible');
  };

  return (
    <Modal ref={modalRef}>
      <button onClick={closeModal}>Close</button>
    </Modal>
  );
}
```

#### ❓ What does `modalRef.current` point to? What does `displayName` affect?

<details>
<summary>✅ Answer</summary>

```txt
modalRef.current: the <div class="modal"> DOM element
displayName: affects how the component appears in React DevTools (shows "Modal" instead of "ForwardRef")
```

**Explanation:** The `ref` is forwarded to the outer `<div>`, so `modalRef.current` is the `<div>` DOM node. `closeModal` correctly removes the CSS class from the real DOM element. `displayName` is a debugging aid — it sets what name appears in the React DevTools component tree. Without it, the component would appear as `ForwardRef` or `ForwardRef(Modal)`. Setting `displayName = 'Modal'` makes the DevTools tree more readable.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function App() {
  const [show, setShow] = useState(false);
  const ref = useRef(null);

  useEffect(() => {
    if (show) {
      console.log('ref after show:', ref.current);
    }
  }, [show]);

  return (
    <>
      <button onClick={() => setShow(true)}>Show</button>
      {show && <p ref={ref}>Visible</p>}
    </>
  );
}
```

#### ❓ After clicking "Show", what does the effect log?

<details>
<summary>✅ Answer</summary>

```txt
ref after show: <p>Visible</p>  (the DOM element)
```

**Explanation:** When `show` changes to `true`, React re-renders with `<p ref={ref}>Visible</p>` in the JSX. React commits the new DOM (including the `<p>`) and sets `ref.current` to the `<p>` element. Then `useEffect` with `[show]` runs, at which point `ref.current` is already the DOM element. React always commits DOM changes (and updates refs) before running effects.

</details>

---

### Q22

```jsx
function App() {
  const ref = useRef({ count: 0, label: 'initial' });

  const update = () => {
    ref.current = { count: ref.current.count + 1, label: 'updated' };
    console.log('ref.current:', ref.current);
  };

  return <button onClick={update}>Update Ref</button>;
}
```

#### ❓ After clicking 3 times, what does the ref hold? Does anything re-render?

<details>
<summary>✅ Answer</summary>

```txt
After 3 clicks: ref.current = { count: 3, label: 'updated' }
No re-renders occur.
```

**Explanation:** `ref.current` can be reassigned to a completely new object — you are not limited to mutating the existing value. The ref container `{ current: ... }` stays the same stable object, but its `.current` property now points to a new object. Each click replaces the entire object and logs the new value. No re-renders happen because mutating `ref.current` never triggers React's rendering cycle. The button label never changes.

</details>

---

### Q23

```jsx
function Child() {
  const ref = useRef(null);

  useEffect(() => {
    console.log('Child mounted, ref:', ref.current);
    return () => {
      console.log('Child unmounting, ref:', ref.current);
    };
  }, []);

  return <div ref={ref}>Child</div>;
}

function App() {
  const [show, setShow] = useState(true);

  return (
    <>
      <button onClick={() => setShow(s => !s)}>Toggle</button>
      {show && <Child />}
    </>
  );
}
```

#### ❓ What is logged when Child mounts and when it unmounts?

<details>
<summary>✅ Answer</summary>

```txt
// On mount:
Child mounted, ref: <div>Child</div>

// On unmount (toggle off):
Child unmounting, ref: null
```

**Explanation:** On mount, the effect runs after DOM commit — `ref.current` is the `<div>`. On unmount, React performs cleanup in this order: (1) calls all cleanup functions, (2) removes DOM elements and resets refs. When the cleanup function runs, `ref.current` is already `null` because React has already detached the ref before running cleanup. This is an important subtlety: you cannot reliably access the DOM node in the cleanup of a `[]`-dep effect — it will already be `null`.

</details>

---

### Q24

```jsx
function Counter() {
  const ref = useRef(0);

  // called every render
  ref.current = ref.current + 1;

  return <p>This component has rendered {ref.current} times</p>;
}

function App() {
  const [x, setX] = useState(0);

  return (
    <>
      <Counter />
      <button onClick={() => setX(v => v + 1)}>Cause parent re-render</button>
    </>
  );
}
```

#### ❓ After clicking the button 3 times, what does the `<p>` inside Counter show?

<details>
<summary>✅ Answer</summary>

```txt
This component has rendered 4 times
```

**Explanation:** `ref.current` is incremented on every render of `Counter`, including the initial render. The parent re-renders on every button click, which causes `Counter` to re-render too (since it is a child). Initial render: 1. Each button click re-renders Counter: +1 each time. After 3 clicks: 4 total renders. Note: the displayed value is correct each time because `ref.current` is incremented before the JSX reads it in the same render.

</details>

---

### Q25

```jsx
function App() {
  const ref1 = useRef(0);
  const ref2 = useRef(0);

  console.log('ref1 === ref2:', ref1 === ref2);
  console.log('ref1.current === ref2.current:', ref1.current === ref2.current);

  ref1.current = 5;

  console.log('after mutation, ref1.current:', ref1.current);
  console.log('after mutation, ref2.current:', ref2.current);

  return <div />;
}
```

#### ❓ What are the four console logs on the initial render?

<details>
<summary>✅ Answer</summary>

```txt
ref1 === ref2: false
ref1.current === ref2.current: true
after mutation, ref1.current: 5
after mutation, ref2.current: 0
```

**Explanation:** Each `useRef(0)` call creates a separate `{ current: 0 }` object. `ref1` and `ref2` are different objects (`ref1 === ref2` is `false`). Their `.current` values start equal at `0` (`ref1.current === ref2.current` is `true`). After `ref1.current = 5`, only `ref1.current` is `5`. `ref2.current` is unchanged at `0`. Refs are independent containers — mutating one ref has no effect on another.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| DOM Access | Q1 – Q5 | null during render, null on unmount, event handler safety, auto-focus on mount, callback ref pattern |
| Mutable Values | Q6 – Q10 | no re-render on mutation, render counter, interval ID storage, usePrevious pattern, inline ref update |
| Stale Closure Fix | Q11 – Q15 | setTimeout stale closure, ref bridge pattern, logLater pattern, useCallback + ref, stable callback with ref |
| forwardRef | Q16 – Q20 | ref forwarding to DOM, warning without forwardRef, useImperativeHandle API restriction, ref not in props, displayName |
| Edge Cases | Q21 – Q25 | ref after conditional render, reassigning ref.current, ref null in cleanup, render counter with parent re-render, ref independence |
