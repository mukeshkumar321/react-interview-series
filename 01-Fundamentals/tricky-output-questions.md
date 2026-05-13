## 📚 React Fundamentals — Tricky Output Questions

> These questions are focused on real React behavior, rendering logic, hooks, JSX, reconciliation, state updates, closures, batching, and edge cases. Each question reflects scenarios you'll encounter in senior React interviews and production code.

---

## 1. JSX & Rendering

### Q1

```jsx
function App() {
  const name = "React";

  return <h1>Hello {name}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
Hello React
```

**Explanation:** JSX expressions enclosed in curly braces `{}` are evaluated as JavaScript code within the component's render context. The variable `name` is in scope and its value `"React"` is evaluated and rendered as a text node. This demonstrates basic JSX interpolation where JavaScript expressions are embedded directly into the JSX markup.

</details>

---

### Q2

```jsx
function App() {
  return <h1>{true}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
(nothing renders — the element is empty)
```

**Explanation:** React treats certain values as "no content" and does not render them to the DOM. The values `true`, `false`, `null`, and `undefined` are all filtered out during the rendering process. This is intentional design to allow developers to conditionally render content using boolean expressions without showing `"true"` or `"false"` strings. This is a key React rendering convention that differs from JavaScript's normal string coercion.

</details>

---

### Q3

```jsx
function App() {
  return <h1>{0}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
0
```

**Explanation:** The number `0` is rendered as visible text on the page. React distinguishes between falsy values — while `false`, `null`, and `undefined` are filtered out, the number `0` is a valid value and renders normally. This distinction exists because `0` is semantically meaningful (it's a real number) whereas `false` and `null` are used to indicate "no content." This is a common source of bugs when using conditional rendering with `count &&` patterns, where `count === 0` will render the `0` instead of nothing.

</details>

---

### Q4

```jsx
function App() {
  const user = {
    name: "John",
  };

  return <h1>{user}</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

```txt
React throws a runtime error:
Objects are not valid as a React child. 
If you meant to render a collection of children, use an array instead.
```

**Explanation:** React forbids rendering plain objects because they are not serializable to DOM strings. The component cannot determine what text representation of the object to display. This safety constraint forces developers to be explicit about what data to render — you must access specific properties like `{user.name}` or convert the object to a string. This prevents accidental rendering of `[object Object]` which was a common problem in other templating libraries.

</details>

---

### Q5

```jsx
function App() {
  return <h1>{["A", "B", "C"]}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
ABC
```

**Explanation:** Arrays of valid renderable content (like strings) are treated specially in React — they are flattened and each element is rendered sequentially without separators. React iterates through the array and renders each string as text. This differs from objects because arrays are iterable and each element is individually renderable. However, this pattern generates a console warning about missing `key` props when used this way. The proper way to render arrays is with `.map()` and explicit keys for reconciliation.

</details>

---

## 2. State Updates

### Q6

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ After one click?

<details>
<summary>✅ Answer</summary>

```txt
1
```

**Explanation:** Both `setCount` calls receive the same stale value of `count` (which is `0` at the time the handler runs). When you call `setCount(count + 1)` twice in the same handler, both calls are batched together but both use the stale closure-captured value. React batches them into a single state update with the value `0 + 1 = 1`. This demonstrates the closure behavior of state updaters — the `count` variable captured in the handler doesn't update until after the handler finishes and React commits the state. To increment twice, use functional updates like `setCount(prev => prev + 1)`.

</details>

---

### Q7

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ After one click?

<details>
<summary>✅ Answer</summary>

```txt
2
```

**Explanation:** Functional state updaters receive the latest queued state value, not the closure-captured value. When you pass a function to `setCount`, React feeds it the current value from the state queue. The first `setCount(prev => prev + 1)` receives `0` and returns `1`, queuing it. The second `setCount(prev => prev + 1)` receives `1` (the result of the first updater) and returns `2`, queuing it. This happens within a single batched update cycle. Functional updates are the correct pattern for state updates that depend on the previous state, especially when multiple updates occur in the same event handler.

</details>

---

### Q8

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  console.log("Render");

  return (
    <button onClick={() => setCount(count)}>
      {count}
    </button>
  );
}
```

#### ❓ Will component re-render?

<details>
<summary>✅ Answer</summary>

```txt
No — the component does not re-render when the button is clicked.
```

**Explanation:** React performs a bailout optimization when the new state value is equal to the current value using `Object.is` comparison. When `setCount(count)` is called with the same value, React checks that the new state (`count`) is identical to the old state (`count`). Since `Object.is(count, count)` is always true, React skips the render phase entirely and does not call the component function again. This is an important optimization to prevent unnecessary renders when state is "updated" to the same value. You'll see "Render" printed only on initial mount, never on click.

</details>

---

## 3. Closures in React

### Q9

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function log() {
    setTimeout(() => {
      console.log(count);
    }, 2000);
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>

      <button onClick={log}>
        Log
      </button>
    </>
  );
}
```

#### ❓ What gets logged?

<details>
<summary>✅ Answer</summary>

```txt
The value of count from the render where log() was called.
If you increment to 5 then click Log, after 2 seconds it logs: 5
If you increment again to 6 before the timeout fires, it still logs: 5
```

**Explanation:** The `log` function and the arrow function passed to `setTimeout` form a closure over the `count` variable from the render where `log()` was invoked. Each time `log` is called, it captures the current value of `count` at that moment. The timeout callback, created inside `log`, closes over that captured `count` value. When the timeout fires 2 seconds later, it reads the closure-captured value, not the current component state. This demonstrates that closures in React capture values, not references — they preserve the state snapshot from when the function was created. This is one of the most common sources of confusion with state and effects in React.

</details>

---

## 4. Event Handling

### Q10

```jsx
function App() {
  function handleClick() {
    console.log("Clicked");
  }

  return <button onClick={handleClick()}>Click</button>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

```txt
The function executes immediately during render.
"Clicked" prints to console during rendering, not on click.
The button onClick receives undefined (the return value of handleClick()).
```

**Explanation:** When you use `onClick={handleClick()}`, the parentheses invoke the function immediately as the component renders. The function runs during the render phase, not in response to user interaction. The return value of `handleClick()` (which is `undefined`) is assigned to the `onClick` prop, so clicking does nothing. The correct syntax is `onClick={handleClick}` (without parentheses) to pass the function itself as an event handler. This is a critical distinction between calling a function and passing a function reference — common mistakes occur when event handlers are accidentally invoked during render.

</details>

---

### Q11

```jsx
<button onClick={() => console.log("Hi")}>
  Click
</button>
```

#### ❓ When does console run?

<details>
<summary>✅ Answer</summary>

```txt
Only after the button is clicked by the user.
```

**Explanation:** The arrow function `() => console.log("Hi")` is not invoked during render — it's passed as a function reference to the `onClick` prop. React stores this function and only calls it when the click event fires. The arrow function creates a new function object on every render (causing an indirect reference change), but it still defers execution until the event occurs. This is the correct pattern for inline event handlers. The arrow function wrapper allows you to execute code and potentially pass arguments or capture variables at the time of the click.

</details>

---

## 5. Conditional Rendering

### Q12

```jsx
function App() {
  return <div>{false && <h1>Hello</h1>}</div>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
<div></div>
(the heading is not rendered)
```

**Explanation:** React uses short-circuit evaluation with logical AND (`&&`). When the left side is `false`, the right side (`<h1>Hello</h1>`) is not evaluated and nothing is rendered. The expression `false && anything` evaluates to `false`, which React filters out as a non-renderable value. This is a common and idiomatic pattern for conditional rendering in React. However, it only works when the left operand is a boolean — if it's a falsy value like `0`, that value renders instead (see Q13).

</details>

---

### Q13

```jsx
function App() {
  return <div>{0 && <h1>Hello</h1>}</div>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
<div>0</div>
(the number 0 is rendered)
```

**Explanation:** While `false && anything` renders nothing, `0 && anything` renders the `0`. This is a subtle but critical distinction. When the left operand of `&&` is a number like `0`, React renders it as valid content rather than filtering it out. This creates a common bug: `{count && <Item />}` renders nothing when `count` is truthy, but renders the literal number `0` when `count` is `0`. The fix is to use a boolean check: `{count > 0 && <Item />}` or `{Boolean(count) && <Item />}`. This demonstrates React's inconsistent treatment of falsy values — `0` is rendered while `false` and `undefined` are not.

</details>

---

## 6. Lists & Keys

### Q14

```jsx
const users = ["A", "B", "C"];

function App() {
  return (
    <>
      {users.map(user => (
        <h1>{user}</h1>
      ))}
    </>
  );
}
```

#### ❓ What warning appears?

<details>
<summary>✅ Answer</summary>

```txt
Warning: Each child in a list should have a unique "key" prop.
```

**Explanation:** React needs a `key` prop on each list item to uniquely identify which items have changed, been added, or removed. Without keys, React falls back to using the array index as the key internally, which can cause bugs if the list is reordered, filtered, or has items inserted/removed. The warning appears in development mode to alert developers. Keys should be stable identifiers (like IDs from data), not array indices, to ensure React correctly matches list items across re-renders and maintains component state correctly for each item.

</details>

---

### Q15

```jsx
{
  users.map((user, index) => (
    <h1 key={index}>{user}</h1>
  ));
}
```

#### ❓ Why can index keys be problematic?

<details>
<summary>✅ Answer</summary>

```txt
If the list is reordered, filtered, or has items inserted/removed, index-based keys cause:
- Component state and DOM elements to be mismatched
- Input focus and values to be lost or misplaced
- Animations and form data to be associated with wrong items
```

**Explanation:** Index keys are stable within a render, but they don't represent the actual identity of the data. If you insert an item at the beginning, all indices shift, and React's reconciliation algorithm (which uses keys to match old and new elements) will incorrectly match new elements to old ones based on index instead of data identity. For example, if an item at index 1 has input state, and a new item is inserted at index 0, the input state moves to index 2 even though it should stay with the original item. This creates hard-to-debug UI bugs. Use stable, unique identifiers from your data (like `key={user.id}`) to ensure React tracks items correctly across structural changes.

</details>

---

## 7. useEffect

### Q16

```jsx
function App() {
  React.useEffect(() => {
    console.log("Effect");
  });

  return <h1>Hello</h1>;
}
```

#### ❓ When does effect run?

<details>
<summary>✅ Answer</summary>

```txt
After every render, including the initial mount.
The console logs "Effect" repeatedly as the component re-renders.
```

**Explanation:** When `useEffect` is called without a dependency array, it runs the effect callback after every render cycle. On initial mount, the component renders, then the effect runs. On every subsequent state or prop change, the component re-renders, and then the effect runs again. This is the least optimized form of `useEffect` and is rarely intended in production code. Most effects should specify a dependency array to control when they run — empty `[]` for mount-only, or `[specificDeps]` to run only when those specific values change. An effect with no dependencies often indicates the developer forgot to add the dependency array.

</details>

---

### Q17

```jsx
React.useEffect(() => {
  console.log("Mounted");
}, []);
```

#### ❓ When does this run?

<details>
<summary>✅ Answer</summary>

```txt
Only once, immediately after the component mounts and the DOM is painted.
```

**Explanation:** An empty dependency array `[]` tells React to run the effect once after the initial render and never again, even if state or props change. This pattern is used for setup tasks that should happen exactly once: fetching initial data, subscribing to events, initializing timers, etc. The effect runs during the "layout" phase after React commits the DOM but before the browser paints. If the component unmounts, any cleanup function returned from the effect will run. This is a fundamental pattern in React for initialization and setup logic.

</details>

---

### Q18

```jsx
React.useEffect(() => {
  console.log("Updated");
}, [count]);
```

#### ❓ When does this run?

<details>
<summary>✅ Answer</summary>

```txt
On the initial render (after mount)
Whenever count changes
Does NOT run if count's value doesn't change (React uses Object.is comparison)
```

**Explanation:** A dependency array with specific values `[count]` tells React to run the effect after mount and after every render where any dependency value has changed. React uses `Object.is` for comparison, so the effect only re-runs if the new `count` value is different from the previous one. If you call `setCount(count)` (setting to the same value), React's bailout optimization prevents re-render entirely, so the effect doesn't run. This is the most common and practical form of `useEffect` for responding to specific state or prop changes. Missing dependencies from the array is a common source of bugs (stale closures), which ESLint's `exhaustive-deps` rule helps catch.

</details>

---

## 8. Infinite Re-render

### Q19

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  setCount(1);

  return <h1>{count}</h1>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

```txt
Infinite re-render loop — React throws a "Too many re-renders" error.
```

**Explanation:** Calling `setCount` directly in the render phase (the component body, not inside an event handler or effect) violates React's rules. When `setCount(1)` is called during render, React schedules a re-render immediately after the current render completes. The component function is called again, `setCount(1)` is called again, another re-render is scheduled, and this cycle repeats until React's internal render limit is exceeded. State updates must only happen in controlled contexts: event handlers, effects, or other callbacks — never directly in the component body. This is one of the most fundamental React rules to prevent infinite loops.

</details>

---

## 9. Controlled Components

### Q20

```jsx
function App() {
  const [value, setValue] = React.useState("");

  return (
    <input
      value={value}
    />
  );
}
```

#### ❓ Can user type?

<details>
<summary>✅ Answer</summary>

```txt
No — the input is frozen at an empty value and cannot be edited.
```

**Explanation:** This is a controlled component because the input's `value` prop is bound to state. However, there's no `onChange` handler to update state when the user types. The input displays the state value (`""`), and when the user tries to type, the DOM input element's value changes temporarily, but React re-renders and resets it back to the state value (`""`). The user's keystrokes are effectively ignored. To make it editable, you must add `onChange={e => setValue(e.target.value)}`. This demonstrates the two-way binding pattern in React: the component controls the input via state, and the input notifies the component of changes via callbacks. Without the callback, the input is write-only from the user's perspective.

</details>

---

## 10. Reconciliation

### Q21

```jsx
function App() {
  return (
    <>
      <h1>Hello</h1>
      <h1>Hello</h1>
    </>
  );
}
```

#### ❓ Does React create full DOM every render?

<details>
<summary>✅ Answer</summary>

```txt
No — React does not recreate the entire DOM tree on every render.
```

**Explanation:** React uses a process called "reconciliation" to efficiently update the DOM. It maintains a Virtual DOM representation of your components and compares ("diffs") the new Virtual DOM with the previous one. React only updates the actual DOM nodes that changed. In this example, even if the component re-renders multiple times, React recognizes that the structure and content remain identical — the two `<h1>` elements with "Hello" text don't change — so the actual DOM is not modified. This reconciliation algorithm is what makes React performant at scale. React's diffing works at the component level (using keys) and the element level (using type and key). Understanding reconciliation is crucial to writing performant React apps and debugging unexpected rendering behavior.

</details>

---

## 11. Strict Mode

### Q22

```jsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

#### ❓ Why might component render twice in development?

<details>
<summary>✅ Answer</summary>

```txt
React Strict Mode intentionally double-invokes the component function and effect callbacks in development mode to detect side effects and bugs.
```

**Explanation:** `React.StrictMode` is a development-only wrapper that intentionally runs components twice: it renders the component, then immediately unmounts and remounts it, causing effects to run twice. This is not a bug — it's intentional to help developers catch common mistakes like effects that lack cleanup, missing dependencies, or side effects in the render phase. If your effect does something and cleans it up correctly, it will be safe to run multiple times. If your component body has side effects (console.log, state updates, API calls), the double-render will make them visible. Strict Mode does not affect production builds. It's a valuable tool for ensuring components are "pure" and resilient to multiple render cycles.

</details>

---

## 12. React.memo

### Q23

```jsx
const Child = React.memo(() => {
  console.log("Child Render");

  return <h1>Child</h1>;
});
```

#### ❓ When does child re-render?

<details>
<summary>✅ Answer</summary>

```txt
Only when its props change (according to shallow comparison).
If the parent re-renders but props to Child stay the same, Child does NOT re-render.
```

**Explanation:** `React.memo` is a higher-order component that wraps a component and prevents re-renders when props haven't changed. It performs a shallow comparison of the previous props object with the new props object. If all props are the same (by `Object.is`), the memoized component returns the cached render result without running the component function again. This optimization prevents unnecessary re-renders of Child when the parent re-renders but doesn't pass new data to Child. However, if any prop changes — even if it's a new function or object with identical content — the memoized component will re-render. `React.memo` is useful for expensive components or long lists, but overusing it can add overhead and obscure performance issues.

</details>

---

## 13. useRef

### Q24

```jsx
function App() {
  const ref = React.useRef(0);

  function update() {
    ref.current++;
    console.log(ref.current);
  }

  return <button onClick={update}>Click</button>;
}
```

#### ❓ Does component re-render?

<details>
<summary>✅ Answer</summary>

```txt
No — the component does not re-render when ref.current is updated.
Clicking logs: 1, 2, 3, 4, ... with no visual change to the button.
```

**Explanation:** `useRef` returns a mutable object `{ current: value }` that persists across renders without triggering re-renders when mutated. Unlike state updates via `setCount`, mutations to `ref.current` do not schedule a re-render. Refs are used for values that don't affect rendering — like storing DOM nodes, tracking timers, keeping mutable instance variables, or storing the previous value of a prop. The ref object itself is stable across renders (same reference), but changing its `.current` property is invisible to React's rendering system. If you need to update state AND see visual changes, use `useState` instead. If you need to track a value without affecting rendering, use `useRef`.

</details>

---

## 14. Batching

### Q25

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setCount(c => c + 1);
    setCount(c => c + 1);

    console.log(count);
  }

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ What gets logged?

<details>
<summary>✅ Answer</summary>

```txt
0
(the old state value from before the click)
```

**Explanation:** React batches multiple state updates within an event handler into a single re-render cycle. The `console.log(count)` runs during the event handler, before React commits the batched updates and before the component re-renders. So `count` still holds the old value (`0`) at the time `console.log` runs. The two `setCount` calls are queued and processed together in a single re-render, resulting in `count` becoming `2`. But the log statement executes synchronously in the handler before any updates commit. This demonstrates that state updates are asynchronous in React — the logged value reflects the state from before the handler, not after. In React 18+, automatic batching applies even to async updates outside event handlers (like in setTimeout or fetch callbacks).

</details>

---

## 15. Fragment

### Q26

```jsx
function App() {
  return (
    <>
      <h1>Hello</h1>
      <h2>World</h2>
    </>
  );
}
```

#### ❓ Why use Fragment?

<details>
<summary>✅ Answer</summary>

```txt
Fragments (`<>...</>`) allow returning multiple elements without creating an unnecessary wrapper DOM node.
```

**Explanation:** In React, a component's render function must return a single root element. Without Fragments, you'd need to wrap multiple elements in a `<div>` or other container. But adding extra wrapper divs clutters the DOM tree and can break CSS layouts (e.g., flexbox with `flex: 1` on direct children). Fragments (`<>...</>` is shorthand for `<React.Fragment>`) solve this by grouping elements without rendering an actual DOM node. The fragment itself doesn't appear in the DOM — only its children do. Fragments are semantic: they express intent that elements are logically grouped but should not create a DOM container. This is especially useful in component composition, lists, and when you need multiple sibling elements but can't add a wrapper.

</details>

---

## 16. Component Rendering

### Q27

```jsx
function Child() {
  console.log("Child");

  return <h1>Child</h1>;
}

function App() {
  const [count, setCount] = React.useState(0);

  return (
    <>
      <Child />
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>
    </>
  );
}
```

#### ❓ Does Child re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — Child re-renders every time App re-renders, even though count doesn't affect Child.
Clicking the button logs "Child" to the console again.
```

**Explanation:** When a parent component re-renders, all of its child components re-render by default unless explicitly optimized with `React.memo`. React doesn't try to be smart about which children "need" to update — it re-renders the whole tree from the parent down. In this case, when `count` changes, `App` re-renders (printing "Child" again in the process), and `Child` is also re-rendered. Since `Child` has no props and is not memoized, React has no reason to skip it. To prevent this re-render, you could wrap `Child` in `React.memo` or restructure to pass `count` only where needed. This is a fundamental source of performance issues in React apps — unnecessary re-renders propagate down the tree, and you must explicitly prevent them.

</details>

---

## 17. Derived State

### Q28

```jsx
function App() {
  const [count] = React.useState(5);

  const doubled = count * 2;

  return <h1>{doubled}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
10
```

**Explanation:** Derived state — values calculated from state — should be computed directly in the component body, not stored in separate state. Here, `doubled` is calculated as `count * 2 = 5 * 2 = 10`. Computing values during render is efficient and keeps data in sync automatically. Every time `count` changes, `doubled` is automatically recalculated on the next render. The only time you should store derived values in state is if the computation is expensive and you want to memoize it (using `useMemo`). Otherwise, computing in the render phase is simpler and less error-prone — no risk of derived state becoming stale or out of sync with its source.

</details>

---

## 18. Boolean Rendering Edge Cases

### Q29

```jsx
function App() {
  return <h1>{NaN}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
NaN
```

**Explanation:** `NaN` (Not-a-Number) is a number type in JavaScript and renders as the string `"NaN"`. Unlike `false`, `null`, and `undefined` which React filters out, `NaN` is treated as a valid renderable value. This is technically correct JavaScript — `typeof NaN === 'number'` — but it's rarely intentional to render `"NaN"` in the UI. This edge case demonstrates React's type-based rendering logic: it filters specific values (`false`, `null`, `undefined`) but allows all other values through, including edge cases like `NaN`. In practice, if you encounter `NaN` rendering, it usually indicates a bug in your calculations or data processing.

</details>

---

### Q30

```jsx
function App() {
  return <h1>{undefined}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```txt
(nothing renders — the element is empty)
```

**Explanation:** `undefined` is one of the special values that React explicitly filters out and does not render. Like `null` and `false`, `undefined` represents "no content." This is useful for optional rendering — if a value is `undefined`, nothing appears on screen rather than the string `"undefined"`. The empty `<h1></h1>` element still exists in the DOM but contains no text node. This design choice allows conditional rendering patterns like `{message}` where `message` might be `undefined` and nothing renders, or a string and the text appears.

</details>

---

## Advanced Questions (Combining Concepts)

### Q31

```jsx
function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setTimeout(() => {
      console.log("Timeout count:", count);
    }, 0);
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

#### ❓ What gets logged after clicking? Explain batching and closures.

<details>
<summary>✅ Answer</summary>

```txt
0
(the count value from before the click)
```

**Explanation:** This question combines state batching with closure behavior. The two `setCount` calls are batched into a single re-render, which would update count to 2. However, the `setTimeout` callback is created before the batching/re-render happens, and it captures `count` from the current render scope via closure. At the time `handleClick` runs, `count` is still `0`, so the closure captures `0`. Even though React batches the two state updates and schedules a re-render that will set count to 2, the `console.log` runs during the same handler execution with the captured closure value. The timeout callback executes after all state updates and re-renders are complete, but it still logs the old value because that's what the closure captured. This demonstrates how closures capture values, not references to state — and how batching doesn't affect what's already captured by callbacks created before the batch.

</details>

---

### Q32

```jsx
const MemoChild = React.memo(({ count, onUpdate }) => {
  console.log("MemoChild render");
  return (
    <button onClick={() => onUpdate(count + 1)}>
      Count: {count}
    </button>
  );
});

function App() {
  const [count, setCount] = React.useState(0);

  return <MemoChild count={count} onUpdate={setCount} />;
}
```

#### ❓ When does MemoChild re-render? Explain memoization, props, and reconciliation.

<details>
<summary>✅ Answer</summary>

```txt
MemoChild re-renders on every click, even though its props logically haven't changed in structure.
Console shows "MemoChild render" on every click.
```

**Explanation:** Although `MemoChild` is wrapped in `React.memo`, the `onUpdate` prop receives a new function reference on every render. The `setCount` function passed to `onUpdate` is recreated or rather re-referenced on every render cycle (it's defined in the App body). React's shallow comparison of props checks if `Object.is(newOnUpdate, oldOnUpdate)` — these are different function references, so `React.memo` considers the props "changed" and does not skip the re-render. To prevent this, you'd need to wrap `setCount` in `useCallback` so it has a stable reference across renders: `useCallback(setCount, [])`. This demonstrates that `React.memo` uses shallow comparison on the props object — new function or object instances always fail the comparison, even if logically they do the same thing. Memoization is not always a silver bullet; you must also ensure the props themselves have stable references for memoization to be effective.

</details>

---

### Q33

```jsx
function App() {
  const [items, setItems] = React.useState(["A", "B", "C"]);

  const handleClick = () => {
    setItems(prev => {
      const newItems = [...prev];
      newItems.unshift("Z");
      return newItems;
    });
  };

  return (
    <div>
      {items.map((item, index) => (
        <input key={index} defaultValue={item} />
      ))}
      <button onClick={handleClick}>Add Z</button>
    </div>
  );
}
```

#### ❓ What happens to the input values when the button is clicked? Explain reconciliation and keys.

<details>
<summary>✅ Answer</summary>

```txt
The input values appear to shift or get corrupted. The first input changes to "Z", 
the second shows "A", the third shows "B", and the fourth (newly added) shows "C".
Any user input typed into the inputs before clicking will be misassociated with different items.
```

**Explanation:** This demonstrates the critical flaw with index-based keys in reconciliation. Initially, items `["A", "B", "C"]` render with keys `[0, 1, 2]`. After clicking, items become `["Z", "A", "B", "C"]`. With index-based keys, React thinks: key 0 now has "Z" (was "A"), key 1 has "A" (was "B"), key 2 has "B" (was "C"), and key 3 is new with "C". React's reconciliation matches elements by key, not by data identity, so it reuses the DOM elements in place. The first `<input key={0}>` with defaultValue "A" now displays "Z" because React thinks that's the same list item. The actual DOM elements aren't recreated (they're reused via reconciliation), but they now represent different data. This is a classic reconciliation bug caused by using indices as keys. The fix is to use stable, unique identifiers: `key={item}` (if items are unique strings) or better yet `key={item.id}` from your data model.

</details>

---

## ✅ Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| JSX & Rendering | Q1 – Q5 | Interpolation, boolean/null/undefined filtering, array flattening, object rejection |
| State Updates | Q6 – Q8 | Stale closures, functional updates, batching, bailout optimization |
| Closures | Q9 | Closure capture, state snapshots, asynchronous callbacks |
| Event Handling | Q10 – Q11 | Immediate invocation vs function reference, event handler timing |
| Conditional Rendering | Q12 – Q13 | Short-circuit evaluation, falsy value edge cases (0 vs false) |
| Lists & Keys | Q14 – Q15 | Key warnings, index vs stable keys, reconciliation bugs |
| useEffect | Q16 – Q18 | No deps vs empty deps, dependency comparison, Object.is |
| Infinite Re-render | Q19 | State updates during render, React limits |
| Controlled Components | Q20 | Two-way binding, onChange requirement |
| Reconciliation | Q21 | Virtual DOM diffing, selective updates |
| Strict Mode | Q22 | Double-invoke in development, side effect detection |
| React.memo | Q23 | Shallow prop comparison, memoization limits |
| useRef | Q24 | Mutable ref, no re-render trigger, use cases |
| Batching | Q25 | Event handler batching, asynchronous state, closure timing |
| Fragment | Q26 | Avoiding wrapper divs, semantic grouping |
| Component Rendering | Q27 | Parent re-renders trigger child re-renders, default behavior |
| Derived State | Q28 | Computed values, render-phase calculation |
| Edge Cases | Q29 – Q30 | NaN rendering, undefined filtering |
| Advanced | Q31 – Q33 | Combining closures + batching, memoization + props, reconciliation + keys |
