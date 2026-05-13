# React Internals Tricky Output Questions

# Table of Contents

1. [Render Cycle Questions](#1-render-cycle-questions)
2. [Batching Questions](#2-batching-questions)
3. [Closure Questions](#3-closure-questions)
4. [useEffect Questions](#4-useeffect-questions)
5. [Reconciliation Questions](#5-reconciliation-questions)
6. [Key Prop Questions](#6-key-prop-questions)
7. [React.memo Questions](#7-reactmemo-questions)
8. [useMemo & useCallback Questions](#8-usememo--usecallback-questions)
9. [StrictMode Questions](#9-strictmode-questions)
10. [Concurrent Rendering Concepts](#10-concurrent-rendering-concepts)
11. [Advanced Fiber & Scheduler Questions](#11-advanced-fiber--scheduler-questions)

---

# 1. Render Cycle Questions

---

## Question 1

```jsx
function Child() {
  console.log("Child Render");

  return <h1>Child</h1>;
}

export default function App() {
  const [count, setCount] = React.useState(0);

  console.log("App Render");

  return (
    <>
      <Child />

      <button
        onClick={() => setCount(count + 1)}
      >
        Increment
      </button>
    </>
  );
}
```

### Output?

<details>
<summary>✅ Answer</summary>

```txt
Initial Render:
App Render
Child Render

After Click:
App Render
Child Render
```

**Explanation:** When a parent component re-renders due to state change, React's render phase causes the component function to re-execute, returning a new React Element tree. During the reconciliation process, React compares the new fiber tree against the previous one; since `Child` is still at the same position in the component hierarchy and has no `React.memo` wrapper, React marks it for re-render. During the commit phase, React invokes the child component's render function to get the updated JSX, and then applies any necessary DOM updates. This is why both parent and child logs appear on each update—React re-executes both component functions during the render phase regardless of whether the child received different props. This demonstrates that without memoization, all descendants are part of the default reconciliation process.

</details>

---

## Question 2

```jsx
function Child() {
  console.log("Child Render");

  return <h1>Child</h1>;
}

const MemoChild = React.memo(Child);

export default function App() {
  const [count, setCount] = React.useState(0);

  console.log("App Render");

  return (
    <>
      <MemoChild />

      <button
        onClick={() => setCount(count + 1)}
      >
        Increment
      </button>
    </>
  );
}
```

### Output?

<details>
<summary>✅ Answer</summary>

```txt
Initial Render:
App Render
Child Render

After Click:
App Render
```

**Explanation:** `React.memo` performs a shallow comparison of the previous props against the new props at the beginning of the render phase. Since `MemoChild` receives no props (or identical props if it did), the comparison passes and React bails out of rendering this component entirely—the render function is never called, and no new fiber is processed. This is a fiber-level optimization; React's reconciler marks the fiber node as "bailed out," skipping both the render phase and the commit phase for that entire subtree. However, note that the parent component still re-renders because the state change originates at the parent level. This illustrates the difference between component render (function execution) and component reconciliation (fiber updating), and why memoization must be applied explicitly to prevent unwanted child renders.

</details>

---

# 2. Batching Questions

---

## Question 3

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
  };

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

### Final State?

<details>
<summary>✅ Answer</summary>

```txt
1
```

**Explanation:** React's automatic batching mechanism groups multiple `setState` calls within a single event handler into a single update cycle. When `setCount(count + 1)` is called twice in quick succession during the same onClick event, both state update actions are queued in the fiber's update queue but not yet merged. During the render phase, React processes the update queue in order; however, both updates capture the same `count` value (0) from the component's closure because state updates are asynchronous and the state hasn't changed yet. The state update queue essentially processes: `setCount(0 + 1)` followed by `setCount(0 + 1)`, which overwrites the first update with the same value (1). This demonstrates the common pitfall of stale closures in event handlers and why functional updaters are preferred. Batching is automatic in event handlers (since React 18), promise handlers, and setTimeout within microtasks, reducing DOM repaints and improving performance.

</details>

---

## Question 4

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const handleClick = () => {
    setCount(c => c + 1);
    setCount(c => c + 1);
  };

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

### Final State?

<details>
<summary>✅ Answer</summary>

```txt
2
```

**Explanation:** Functional state updates (updater functions) fundamentally change how batching works by allowing each update to read the result of the previous update within the same batch. When `setCount(c => c + 1)` is called, React queues the updater function itself, not the immediately computed value. During the render phase, as React processes the update queue, it applies the first updater: the previous state (0) is passed to the function, returning 1, which updates the fiber's state hook. The next updater then receives this updated state (1), applies the function, and returns 2. This queue-merging process with functional updaters is the recommended pattern when multiple updates depend on the previous state. The scheduler's batching mechanism doesn't affect the ordering here—functional updaters ensure that each update composes correctly. This pattern is essential for predictable state management in rapid-fire updates and is a core principle that many state management libraries are built upon.

</details>

---

## Question 5

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  console.log("Render", count);

  return (
    <button
      onClick={() => {
        setCount(c => c + 1);
        setCount(c => c + 1);

        console.log(count);
      }}
    >
      Click
    </button>
  );
}
```

### Output After Click?

<details>
<summary>✅ Answer</summary>

```txt
0
Render 2
```

**Explanation:** State updates are asynchronous; they don't mutate the component's captured state variable immediately. Inside the event handler, both `setCount` calls queue updates in the fiber's update queue and enqueue a re-render via the scheduler, but they don't update the local `count` variable in the current function scope. The `console.log(count)` executes synchronously while still holding the old closure value (0), demonstrating stale closures—a common source of bugs in React. After the event handler completes, React processes the batched updates via the scheduler's work loop, applying both functional updaters to compute the new state (2), then re-invoking the component function with the new state value. This second render generates the "Render 2" log, illustrating the two-phase nature of state management: the synchronous effect phase where updates are queued, and the asynchronous reconciliation phase where renders occur. Understanding this distinction is critical for debugging state-related race conditions and timing issues in React applications.

</details>

---

# 3. Closure Questions

---

## Question 6

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  function handleClick() {
    setTimeout(() => {
      console.log(count);
    }, 1000);
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>

      <button onClick={handleClick}>
        Log
      </button>
    </>
  );
}
```

### What Gets Logged?

<details>
<summary>✅ Answer</summary>

```txt
If clicked at count=0:
0

If "Increment" clicked 5 times, then "Log":
0

Explanation: The stale value from when Log button was clicked, regardless of subsequent state changes
```

**Explanation:** JavaScript closures capture variables from their enclosing lexical scope at definition time, not at execution time. When `handleClick` is called, it creates a closure over the current `count` value. The `setTimeout` callback forms another closure, capturing `handleClick`'s scope. Even though `count` may update many times and trigger multiple re-renders (each creating a new component instance with a new `count` value), the `setTimeout` callback still references the original `count` value from its creation time. React's functional component model exacerbates this because each render creates a new function instance with new closures—this is both a powerful feature (allowing encapsulation) and a pitfall (enabling stale closures). This pattern appears everywhere in React: event handlers, effects without proper dependencies, and callbacks passed to children. The fix involves using the functional updater form (`setCount(c => c + 1)`) or ensuring effect dependencies are correctly specified to avoid stale references. This closure behavior is fundamental to understanding why React's dependency array exists and why many React developers struggle with timing issues.

</details>

---

## Question 7

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    console.log(count);
  }, []);

  return (
    <button
      onClick={() => setCount(count + 1)}
    >
      {count}
    </button>
  );
}
```

### Output?

<details>
<summary>✅ Answer</summary>

```txt
0
```

**Explanation:** The dependency array (`[]`) is a signal to React's scheduler to run this effect only once, after the initial commit phase of the component's mount. React stores the dependency array values internally on the effect hook and compares them during subsequent renders using shallow equality (Object.is for primitives, reference equality for objects). Since the array is empty, there are no dependencies to check, so React always considers the effect as having the same dependencies and skips re-running it. The effect captures the `count` value from the first render (0) in its closure. Even though the button's onClick handler triggers multiple re-renders with updated `count` values, the effect cleanup and rerun logic is bypassed because the dependency comparison fails to detect any changes. This is a classic stale closure problem within effects and demonstrates why missing dependencies in the dependency array is a common source of bugs. React's ESLint plugin (`eslint-plugin-react-hooks`) flags this pattern to help developers catch these errors, as an empty dependency array signals either "run once on mount" or potentially a missed dependency.

</details>

---

# 4. useEffect Questions

---

## Question 8

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    console.log("Effect");

    return () => {
      console.log("Cleanup");
    };
  }, [count]);

  return (
    <button
      onClick={() => setCount(count + 1)}
    >
      {count}
    </button>
  );
}
```

### Output After Click?

<details>
<summary>✅ Answer</summary>

```txt
Cleanup
Effect
```

**Explanation:** Effects run after the commit phase completes and the browser has painted. When the dependency array `[count]` changes, React schedules a microtask (via the scheduler) to run the cleanup function followed by the new effect function. During the effect cleanup phase, React calls the returned cleanup function from the previous effect, executing the logged "Cleanup." After cleanup, React immediately invokes the new effect function, logging "Effect." This process is called "effect chaining" and is crucial for preventing memory leaks; cleanup functions allow you to cancel subscriptions, remove event listeners, and clear timers that were set up in the previous effect. React's fiber architecture tracks which effects need cleanup by storing the cleanup function reference on the fiber node. The order matters: React always cleans up the old effect before running the new one to avoid stale closures in the cleanup function. This is why subscribers to the same resource won't accumulate memory during multiple renders—each effect run is paired with a cleanup run. Understanding this lifecycle is essential for working with side effects in React and preventing common bugs like multiple subscriptions or event listener leaks.

</details>

---

## Question 9

```jsx
export default function App() {
  console.log("Render");

  React.useEffect(() => {
    console.log("Effect");
  });

  return <h1>Hello</h1>;
}
```

### Output Order?

<details>
<summary>✅ Answer</summary>

```txt
Render
Effect
```

**Explanation:** React's rendering process is divided into two distinct phases: the render phase and the commit phase. During the render phase, React calls the component function to get JSX, creates or updates the fiber tree, and performs reconciliation without touching the DOM. Once the render phase completes, React enters the commit phase, where it applies all accumulated DOM changes and flushes effect callbacks. Effects are intentionally deferred until after commit to ensure the DOM is in a consistent state—this prevents effects from reading stale DOM values or causing infinite loops by modifying the DOM during the render phase. When no dependency array is provided (as in this question), React runs the effect after every render. The scheduler orchestrates this timing: after the commit phase, effects are queued as microtasks and executed as soon as possible (but still after all synchronous code in the component has completed). This separation of concerns ensures predictable behavior and prevents effects from interfering with React's internal state management. Many React bugs stem from misunderstanding this timing; for instance, trying to read a DOM ref value during render won't work because the DOM hasn't been updated yet.

</details>

---

# 5. Reconciliation Questions

---

## Question 10

```jsx
export default function App() {
  const [toggle, setToggle] = React.useState(false);

  return (
    <>
      {toggle ? <h1>Hello</h1> : <div>Hello</div>}

      <button onClick={() => setToggle(!toggle)}>
        Toggle
      </button>
    </>
  );
}
```

### What Happens Internally?

<details>
<summary>✅ Answer</summary>

```txt
Entire subtree destroyed and recreated
```

**Explanation:** React's reconciliation algorithm uses element type as a primary heuristic for determining whether to reuse or replace a DOM node. When toggling between `<h1>` and `<div>`, the element types differ fundamentally, so React's diffing algorithm (part of the scheduler's render phase) marks the old fiber for unmounting and creates a new fiber with the different type. During the commit phase, React removes all DOM nodes belonging to the old subtree (destroying any internal state, event listeners, and refs), then creates and inserts new DOM nodes for the new element type. This is an expensive operation but necessary for correctness: you cannot safely mutate an `h1` element into a `div` element in the DOM tree because they have different attributes, event properties, and rendering characteristics. React deliberately destroys and recreates the subtree to ensure no stale state from the previous element persists. This is why conditionally rendering different component types on the same position will reset all nested component state, internal refs, and form values—a gotcha that confuses many React developers. To preserve state across conditional rendering, extract the conditional logic inside the component itself or use a wrapping component with a stable key. Understanding this principle is fundamental to mastering React's reconciliation and avoiding unexpected state resets.

</details>

---

## Question 11

```jsx
function Input() {
  const [value, setValue] = React.useState("");

  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value)}
    />
  );
}

export default function App() {
  const [toggle, setToggle] = React.useState(false);

  return (
    <>
      {toggle ? <Input /> : <Input />}

      <button onClick={() => setToggle(!toggle)}>
        Toggle
      </button>
    </>
  );
}
```

### Will Input State Reset?

<details>
<summary>✅ Answer</summary>

```txt
No, the input state persists
```

**Explanation:** Although both branches render the same `Input` component, React's reconciliation algorithm uses component position (index) in the component tree to identify and reuse fibers, not the source code location. When toggling between the conditional branches, React sees the same component type (`Input`) at the same position in the tree, so it considers these two render outputs as the same logical component and reuses the existing fiber node. The fiber already has an internal hooks queue with the `useState` state stored on it, and since the fiber is reused rather than destroyed, the state persists across the conditional toggle. This is the "same component type, same position" rule in action and demonstrates that React's reconciliation doesn't deeply inspect JSX—it only compares element types and positions. However, if the JSX were structured as `toggle ? <Input /> : <div><Input /></div>`, the Input position would change in the tree and the state would reset because the fiber would be unmounted and remounted. This nuance is crucial for understanding when React preserves component state and when it resets it, and it's why many developers accidentally lose state when refactoring their JSX structure. This principle also explains why using array indices as keys in lists is dangerous: if the list reorders, the same position now corresponds to different data, causing state to move to the wrong items.

</details>

---

# 6. Key Prop Questions

---

## Question 12

```jsx
{
  items.map((item, index) => (
    <Input key={index} />
  ));
}
```

### Why Is This Dangerous?

<details>
<summary>✅ Answer</summary>

```txt
Keys should be stable identifiers, not indices
State will move to wrong items when list changes
```

**Explanation:** The `key` prop is React's signal to the reconciliation algorithm about whether a fiber should be preserved or recreated. When using array indices as keys, the key changes when the array order changes, insertion occurs, or deletion occurs—these are exactly the scenarios where you'd want to preserve component state, but index keys do the opposite. Consider a list of inputs where each item renders an `<Input key={index} />` component with internal state. If you delete the first item, the old second item (previously at index 1) is now at index 0, giving it the same key as the deleted item. React's reconciliation sees the same key and reuses the fiber, but the input's state now corresponds to different data. During re-renders, if a key changes, React treats it as a new component and unmounts the old one, losing any internal state. Additionally, if items are inserted at the beginning, all existing indices shift, causing massive unnecessary unmounts and remounts. Index keys defeat React's ability to track which component instance corresponds to which data item across renders. The correct pattern is using unique, stable identifiers from the data itself (like database IDs or UUIDs), which remain constant regardless of list order, insertion, or deletion. This is a pitfall that wastes performance and causes subtle bugs that are hard to reproduce.

</details>

---

## Question 13

```jsx
{
  items.map(item => (
    <Input key={Math.random()} />
  ));
}
```

### What Happens?

<details>
<summary>✅ Answer</summary>

```txt
All components remount on every render
State resets every time the component rerenders
Performance degradation and state loss
```

**Explanation:** Since `Math.random()` generates a new random value on every render, the `key` prop becomes different on each render. React compares keys during the render phase's reconciliation process; when a key changes, React's reconciliation algorithm treats it as a completely different component and unmounts the old fiber, destroying its internal state, effect cleanups, and refs. Then it creates a new fiber and mounts it fresh. With a random key, this unmount/remount cycle happens on every single render of the parent component, regardless of whether the data changed. This causes several problems: (1) component state is constantly lost, (2) effect cleanups run unnecessarily, causing memory leaks if not careful, (3) DOM nodes are constantly destroyed and recreated, causing the browser to re-parse CSS and repaint unnecessarily, and (4) form inputs lose focus and input values. This pattern is one of the most harmful anti-patterns in React and completely defeats the purpose of keys. React's profiler can detect this by showing constant component unmount/mount cycles. Even `<div key={Math.random()}>` will cause unnecessary DOM thrashing. The solution is always to use stable identifiers from your data source. Understanding this pattern's dangers is crucial because it's a mistake that seems innocuous but has cascading negative effects on application performance and correctness.

</details>

---

# 7. React.memo Questions

---

## Question 14

```jsx
const Child = React.memo(({ obj }) => {
  console.log("Child");

  return <h1>Child</h1>;
});

export default function App() {
  const [count, setCount] = React.useState(0);

  return (
    <>
      <Child obj={{ name: "React" }} />

      <button
        onClick={() => setCount(count + 1)}
      >
        Click
      </button>
    </>
  );
}
```

### Will Child Re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes, Child will re-render on every click
```

**Explanation:** `React.memo` performs shallow equality checks on props, comparing each prop value using the same-reference (`===`) comparison for objects. In this code, the expression `{ name: "React" }` creates a brand new object literal on every render of the parent component. Although the object has the same property values each time, it's a different object in memory—the reference changes every render. When React's reconciliation algorithm runs the shallow prop comparison during the render phase, it compares the old object reference against the new object reference, finds they're different, and invalidates the memoization, causing the child to re-render. This is a critical subtlety in React optimization: memoization only works when prop values don't change, but object literals are recreated on every render. The solution is to move the object outside the render scope using `useMemo` on the parent side, or to pass the individual values as separate props to the child. This pattern is a common performance optimization pitfall—developers add `React.memo` expecting dramatic performance improvements, but accidentally create new object references on every render, nullifying the memoization entirely. Understanding shallow vs. deep equality and reference semantics is essential for effective memoization. Many state management libraries and performance profilers exist primarily because developers struggle with this concept. React's `useCallback` and `useMemo` hooks were created specifically to help manage object and function reference stability.

</details>

---

# 8. useMemo & useCallback Questions

---

## Question 15

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const data = React.useMemo(() => {
    return { name: "React" };
  }, []);

  console.log(data);

  return (
    <button
      onClick={() => setCount(count + 1)}
    >
      Click
    </button>
  );
}
```

### Does Object Reference Change?

<details>
<summary>✅ Answer</summary>

```txt
No, the same object reference is preserved across renders
```

**Explanation:** `useMemo` is a hook that memoizes a computed value based on a dependency array, preventing unnecessary recalculation and preserving object references. React stores the memoized value and dependency array on the fiber's hooks queue. During the render phase, before the component function executes, React checks the dependency array; if it matches the previous array (using shallow equality), React returns the cached value from the previous render without re-invoking the callback. In this example, the dependency array is empty (`[]`), meaning the dependencies never change, so React always returns the same object reference that was created during the initial render. This is why the same object identity persists across all subsequent re-renders caused by the state update. The memoized value is stored as a closure in the hook and retrieved on every render without re-executing the function body. This is particularly useful for expensive computations (like heavy data transformations or calculations) and for maintaining stable object references to prevent unnecessary child re-renders when the object is passed as a prop. However, `useMemo` has overhead—React must still maintain the hook state and perform dependency comparisons—so it shouldn't be overused. It's most beneficial when the computation is genuinely expensive or when the memoized value is passed to memoized children. Understanding when and when not to use `useMemo` is part of advanced performance optimization.

</details>

---

## Question 16

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  const fn = React.useCallback(() => {
    console.log(count);
  }, []);

  return (
    <button onClick={fn}>
      Click
    </button>
  );
}
```

### What Gets Logged After Multiple Updates?

<details>
<summary>✅ Answer</summary>

```txt
0
```

**Explanation:** `useCallback` memoizes a function based on a dependency array, similar to `useMemo` but specifically for functions. React stores the function closure on the fiber's hooks queue; when the dependency array hasn't changed (using shallow equality), React returns the same function instance that was created during an earlier render rather than creating a new one. In this example, the dependency array is empty, so the callback function is created once during the initial render and never recreated. The returned function is the same function object across all renders, and its closure captures the `count` value from the initial render (0). When the button is clicked and state updates, even though `count` changes in the component's state, the callback function still closes over the stale `count` value. The callback's closure was created during the first render and "frozen" by memoization; it doesn't get updated with new state values. This is a stale closure problem amplified by `useCallback`: the hook prevents the function from being recreated, which would have given it a new closure with the latest state. The solution is to include `count` in the dependency array: `useCallback(() => { console.log(count); }, [count])`, which would cause React to recreate the function whenever `count` changes, giving it a fresh closure. This pattern teaches an important lesson: memoization isn't always beneficial—sometimes recreating the function with a new closure is necessary for correctness. The `eslint-plugin-react-hooks` plugin can help catch these issues.

</details>

---

# 9. StrictMode Questions

---

## Question 17

```jsx
export default function App() {
  React.useEffect(() => {
    console.log("Effect");
  }, []);

  return <h1>Hello</h1>;
}
```

### Why Does Effect Run Twice In Development?

<details>
<summary>✅ Answer</summary>

```txt
React.StrictMode intentionally double-invokes effects in development mode
```

**Explanation:** `React.StrictMode` is a development-only feature that intentionally triggers additional checks and warnings to help developers identify potential bugs. For effects, React deliberately runs the effect and its cleanup function twice during development: once during the first render, then immediately unmounts, cleans up, and re-mounts the component to run the effect again. This pattern reveals whether your effect cleanup properly cancels side effects; if the effect has a bug like creating multiple subscriptions or event listeners that don't fully clean up, running it twice will expose this problem. The double-invocation happens during the render phase; React queues the effect, runs cleanup, and re-queues it in rapid succession to simulate an unmount-remount scenario that could happen in production if the component is removed from the tree and added back. This is NOT a bug in your code—it's intentional testing behavior. In production builds, effects run only once as normal. The pattern teaches an important lesson about effect cleanups: they should be properly implemented to handle re-running multiple times without side effects. Many developers are confused by this behavior and think they have a bug when they don't; disabling StrictMode to "fix" the double-invocation is a common mistake that masks real problems. Properly written effects should pass StrictMode's double-invocation test. Understanding this behavior is crucial for writing robust, production-ready React code.

</details>

---

## Question 18

```jsx
export default function App() {
  console.log("Render");

  return <h1>Hello</h1>;
}
```

### Why Can Render Run Twice In Development?

<details>
<summary>✅ Answer</summary>

```txt
StrictMode double-renders components to detect impure render functions
```

**Explanation:** Beyond effects, `React.StrictMode` also double-invokes the component function itself during the render phase in development. React's rendering should be a pure function: given the same input (props and state), it should return the same output without side effects. If a component's render function has side effects like modifying external state, logging (unless for debugging), or calling APIs, this violates React's purity contract and can cause subtle bugs, especially with concurrent rendering. By intentionally rendering the component twice, StrictMode exposes impure render functions: if the component logs something or modifies external state on each render, you'll see it happen twice, making the impurity obvious. This double-rendering happens during the render phase (not affecting the DOM), and React discards the results of the first render and uses only the second one for the commit phase. The pattern teaches that component functions must be pure: they should only return JSX and compute values based on props and state, never perform I/O or side effects. In production, the double-render is removed, so side effects in render functions will only occur once, but they're still bugs that can cause inconsistent behavior. Understanding this distinction between the render phase (which should be pure and repeatable) and the commit phase (where side effects are safe) is fundamental to writing correct React code. Many bugs in React applications stem from accidentally placing side-effectful code in the render phase instead of in effects.

</details>

---

# 10. Concurrent Rendering Concepts

---

## Question 19

```jsx
startTransition(() => {
  setSearchResults(data);
});

setInputValue(value);
```

### Which Update Gets Priority?

<details>
<summary>✅ Answer</summary>

```txt
setInputValue gets priority
Transition updates are lower priority
```

**Explanation:** React's scheduler maintains multiple priority levels for updates, with different urgency tiers. Updates made via `startTransition` are marked as "Transition" priority (lower priority), while regular `setState` calls like `setInputValue` are marked as "Urgent" or "User-blocking" priority (higher priority). The scheduler's work loop processes queued updates in priority order: urgent updates are always processed before transition updates. This allows React to keep the UI responsive to user input—if a user types in an input field while a heavy search result update is processing, React will pause the search computation and immediately process the input update so the typing appears responsive. After the urgent update commits and the component re-renders, the scheduler returns to processing the transition update. This is the core mechanism of concurrent rendering: instead of blocking the main thread on long computations, React can pause in the middle of rendering and give the browser a chance to process user input, animations, and other tasks. The `useTransition` hook provides a way to track whether a transition is pending, useful for showing loading states. Under the hood, the scheduler uses cooperative scheduling: during the render phase, React periodically checks if higher-priority work has arrived and if so, pauses the current render, saves its progress in the fiber tree, and switches to the higher-priority work. This is why React is called "concurrent" even though JavaScript remains single-threaded; React concurrently interleaves work at different priority levels. Understanding the scheduler's priority system is key to performance optimization in large applications.

</details>

---

## Question 20

```jsx
function App() {
  while (true) {}

  return <h1>Hello</h1>;
}
```

### Can Concurrent Rendering Save This?

<details>
<summary>✅ Answer</summary>

```txt
No, concurrent rendering cannot prevent this
```

**Explanation:** Although concurrent rendering and the scheduler are powerful tools for keeping React responsive, they operate within the JavaScript execution model, which remains fundamentally single-threaded. When a component's render function contains an infinite loop (`while (true) {}`), the JavaScript engine never exits the component function; it blocks the main thread indefinitely before React can save progress or pause the render. Concurrent rendering's ability to pause and resume only works between logical pause points, such as after processing a batch of work in the fiber tree. If the render function itself never completes, React never reaches a pause point and cannot hand control back to the browser. This infinite loop would freeze the entire application, preventing all user input, animations, and other JavaScript from executing. The scheduler's work loop is invoked after the component function returns, so it cannot interrupt a blocked component function. This is why careful code review and testing are essential: no amount of framework sophistication can protect against bugs in user code, particularly synchronous infinite loops or extremely long-running computations in render functions. The lesson here is that render functions must complete quickly (typically in milliseconds) and shouldn't perform heavy computations or I/O. For CPU-intensive tasks, use Web Workers; for I/O, use effects and move the work outside the render phase. This is a humbling reminder that even sophisticated systems like React's concurrent scheduler have fundamental limitations imposed by the JavaScript runtime.

</details>

---

# 11. Advanced Fiber & Scheduler Questions

---

## Question 21: Scheduler + Batching + Reconciliation

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);
  const [name, setName] = React.useState("React");

  console.log("Render", count, name);

  return (
    <div>
      <button onClick={() => {
        setCount(c => c + 1);
        setName("Next.js");
        console.log("After setState");
      }}>
        Update
      </button>
      <p>{count} {name}</p>
    </div>
  );
}
```

### What Is The Exact Output?

<details>
<summary>✅ Answer</summary>

```txt
After setState
Render 1 Next.js
```

**Explanation:** This question combines three key React concepts: batching, the scheduler, and multiple state updates in a single event handler. When the onClick handler executes, both `setCount` and `setName` are called, queuing state updates on the fiber's hook queue. React's automatic batching (introduced in React 18) groups these into a single scheduler task. The `console.log("After setState")` executes immediately while still in the event handler, using the old closure values, demonstrating that state updates are asynchronous and batched. After the event handler completes, the scheduler's work loop processes the single batch: React begins the render phase, invoking the reconciliation algorithm to create a new fiber tree with the updated state values. During reconciliation, React compares the old fiber tree with the new one; since the JSX structure is identical (only the state values changed), React updates the fiber nodes in-place rather than replacing them. The component function is re-invoked, logging "Render 1 Next.js" with both state values updated. The scheduler then enters the commit phase, applying DOM updates (in this case, updating the paragraph text). This entire process—queueing, batching, rendering, reconciling, and committing—happens as one unit of work in the scheduler, ensuring that the DOM is never left in an inconsistent state. Understanding this flow is critical for debugging complex interactions involving multiple state updates.

</details>

---

## Question 22: Fiber Reconciliation + Key Prop

```jsx
function Item({ id }) {
  const [count, setCount] = React.useState(0);

  return (
    <div>
      <p>Item {id}: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>
        +
      </button>
    </div>
  );
}

export default function App() {
  const [items, setItems] = React.useState([1, 2, 3]);

  return (
    <div>
      {items.map(id => <Item key={id} id={id} />)}
      <button onClick={() => {
        setItems([4, ...items]);
      }}>
        Prepend
      </button>
    </div>
  );
}
```

### What Happens To Component State After Prepending?

<details>
<summary>✅ Answer</summary>

```txt
Component state is preserved correctly
Item 4 has count: 0
Item 1 has count: 1 (its previous count)
Item 2 has count: 2 (its previous count)
```

**Explanation:** This demonstrates why stable, unique keys are crucial for React's fiber reconciliation. Using `key={id}` (where `id` is a stable identifier from the data) allows React to correctly track each component instance across re-renders. When items are prepended and the array becomes `[4, 1, 2, 3]`, React's reconciliation algorithm uses keys to match fibers: the fiber with `key={1}` is still associated with the Item component with `id={1}`, and its internal state (the `count` from `useState`) is preserved. If instead the code used `key={index}` (the common pitfall), prepending would shift all indices: the old Item with `id={1}` at index 0 would now be at index 1, and the reconciliation algorithm would incorrectly reuse the fiber at index 0 for the new Item with `id={4}`. The state would move to the wrong item, causing confusing bugs where the count appears to move between items. The fiber tree contains nodes with metadata about which component instance each node represents; keys enable React to correlate fiber nodes from the previous tree to the new tree. During reconciliation, React builds a map of keys to fibers, then attempts to match fibers from the new tree to the old tree using keys. Matched fibers are updated in-place; unmatched fibers are created or destroyed. This is why correct key usage is so critical and why the React team warns against index keys in documentation and linters. Understanding fiber reconciliation at this level helps prevent entire classes of state-related bugs.

</details>

---

## Question 23: Effect Batching + Cleanup + Scheduler Priority

```jsx
export default function App() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    console.log("Effect", count);

    return () => {
      console.log("Cleanup", count);
    };
  }, [count]);

  React.useEffect(() => {
    startTransition(() => {
      console.log("Transition effect", count);
    });
  }, [count]);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      {count}
    </button>
  );
}
```

### What Is The Complete Output Order On Initial Render And After Click?

<details>
<summary>✅ Answer</summary>

```txt
Initial Render:
Effect 0
Transition effect 0

After Click (assuming startTransition is used):
Cleanup 0
Effect 1
Transition effect 1
```

**Explanation:** This question tests understanding of effect ordering, batching, and the scheduler's priority system. On initial mount, both effects run after the commit phase in the order they're defined (top to bottom). The first effect logs "Effect 0" and sets up cleanup. The second effect logs "Transition effect 0"—note that `startTransition` only affects the priority of state updates, not effect scheduling, so the transition effect runs immediately after the commit, not deferred. After clicking the button, the state updates, which batches a single render via the scheduler. During the render phase, both effects' dependency arrays change (count changed from 0 to 1), so both effects are marked for cleanup and re-run. React processes the commit phase: first, all cleanups run in reverse order of definition (LIFO for effects), so "Cleanup 0" executes. Then all new effects run in definition order: "Effect 1" logs, followed by "Transition effect 1". The `startTransition` call inside the second effect doesn't change the timing of the effect itself (effects always run synchronously after commit), only the priority of any state updates triggered inside it. If the transition effect triggered a state update, that update would be deferred to lower priority than the current click's update. Understanding the interplay between effect cleanup, multiple effects, and the scheduler is complex but crucial for building applications with multiple coordinated effects and side effects. This pattern tests mastery of React's internal timing and priority systems.

</details>

---

# Important Internal Concepts Tested

These questions comprehensively test:

- Render cycle and fiber reconciliation
- Automatic batching and state update queueing
- Stale closures and closure semantics
- Effect timing, cleanup, and ordering
- Reconciliation diffing and element types
- Key prop and list reconciliation
- React.memo shallow equality and reference semantics
- useMemo and useCallback memoization and dependency arrays
- StrictMode double-invocation for testing purity
- Concurrent rendering and scheduler priorities
- Fiber identity tracking and state preservation
- Complex interactions of multiple systems

---

# Final Mental Model

To master React Internals, always think in terms of this architecture:

```txt
User Interaction / External Event
 ↓
State Update Queued on Fiber
 ↓
Scheduler (Priority System)
 ↓
Render Phase
  ├─ Component Function Called
  ├─ JSX → Fiber Tree
  ├─ Reconciliation (Diffing)
  └─ Dependency Array Checks for Hooks
 ↓
Commit Phase
  ├─ DOM Updates Applied
  ├─ Refs Updated
  ├─ Layout Effects Run
  └─ Effects Scheduled as Microtasks
 ↓
Browser Paint & Event Loop
 ↓
Effects Executed as Microtasks
 ↓
Component Interactive Again
```

**Key Takeaways:**

1. **Fiber Tree:** React maintains a parallel fiber tree (internal representation) that's updated during render, not JSX. Fibers store component state, hooks, and identity information.

2. **Reconciliation:** React's diffing algorithm uses element type and keys to match fibers from old tree to new tree. Mismatched types or keys cause unmount/remount cycles.

3. **Scheduler:** React queues work at priority levels (Urgent, Transition, etc.) and can pause/resume work between logical units. This enables concurrent rendering despite JavaScript being single-threaded.

4. **Batching:** Multiple state updates in the same event handler are automatically batched into one render-commit cycle, optimizing performance.

5. **Closures & Stale Values:** Component functions are called on every render, creating new closures. Without proper dependency tracking (effects, useCallback, useMemo), code can reference stale values.

6. **Effects:** Run after commit phase, scheduled as microtasks. Always include dependency array to avoid stale closures. Cleanup functions prevent memory leaks.

7. **Memoization:** React.memo, useMemo, and useCallback prevent re-execution but only work when references are stable. Creating new objects/functions defeats memoization.

8. **Testing & Development:** StrictMode intentionally triggers extra behavior (double-rendering, double-effect-invocation) to catch impure code and cleanup bugs.

Understanding these concepts deeply allows you to predict React's behavior in complex scenarios, optimize performance correctly, and avoid entire classes of bugs.
