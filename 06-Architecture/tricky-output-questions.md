## React Architecture — Tricky Output Questions

> These questions test your understanding of HOC behavior, render props execution, compound component context, unidirectional data flow, error boundary catch rules, Suspense fallback timing, and memoization edge cases. Each scenario reflects a real architectural interview question or production bug.

---

## 1. Component Patterns

### Q1

```jsx
function withLogger(WrappedComponent) {
  return function LoggedComponent(props) {
    console.log("Rendering:", WrappedComponent.name);
    return <WrappedComponent {...props} />;
  };
}

const LoggedButton = withLogger(Button);
const LoggedInput = withLogger(Input);

function App() {
  return (
    <div>
      <LoggedButton label="Click" />
      <LoggedInput value="test" />
    </div>
  );
}
```

#### ❓ What is logged when App renders?

<details>
<summary>✅ Answer</summary>

```txt
Rendering: Button
Rendering: Input
```

**Explanation:** Each HOC wraps the original component and logs its `name` property. `WrappedComponent.name` is the function name of the original component — `"Button"` and `"Input"` respectively. The HOC renders are transparent: `LoggedButton` renders by invoking `LoggedComponent`, which logs `WrappedComponent.name` and then renders `Button`. Same for `LoggedInput`. This is normal HOC behavior. Note: if the component is anonymous (arrow function assigned to a variable), `name` would reflect the variable name due to ES6 inference.

</details>

---

### Q2

```jsx
function withLogger(WrappedComponent) {
  return (props) => {
    console.log("Rendering");
    return <WrappedComponent {...props} />;
  };
}

const LoggedButton = withLogger(Button);

function App() {
  return <LoggedButton label="Click" />;
}
```

#### ❓ In React DevTools, how does the component tree appear for the HOC-wrapped button?

<details>
<summary>✅ Answer</summary>

```txt
Without displayName set:
  App
    Component    ← anonymous function, shown as "Component" or unnamed
      Button

With displayName set properly:
  App
    withLogger(Button)
      Button
```

**Explanation:** When the HOC returns an anonymous arrow function, React DevTools shows it as `Component` or an unnamed component, making debugging difficult. The fix is to set `displayName` on the returned component: `const LoggedComponent = (props) => ...; LoggedComponent.displayName = \`withLogger(\${WrappedComponent.displayName || WrappedComponent.name})\`;`. This gives DevTools the readable name `withLogger(Button)`.

</details>

---

### Q3

```jsx
function DataFetcher({ url, children }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, [url]);

  return children(data);
}

function App() {
  return (
    <DataFetcher url="/api/user">
      {(user) => (
        user ? <p>{user.name}</p> : <p>Loading...</p>
      )}
    </DataFetcher>
  );
}
```

#### ❓ What renders initially? What renders after the fetch completes?

<details>
<summary>✅ Answer</summary>

```txt
Initially: "Loading..."
After fetch: user.name (e.g., "Alice")
```

**Explanation:** This is the render props pattern where `children` is a function. Initially, `data` is `null`. `DataFetcher` calls `children(null)` — the function receives `null`, evaluates `null ? ... : <p>Loading...</p>` and renders "Loading...". After the fetch completes, `setData(user)` triggers a re-render. `children(user)` is called with the user object. The condition is truthy, so `<p>{user.name}</p>` renders. The render prop pattern inverts control — `DataFetcher` handles data fetching but the caller decides how to render it.

</details>

---

### Q4

```jsx
const SelectContext = createContext();

function Select({ children, value, onChange }) {
  return (
    <SelectContext.Provider value={{ value, onChange }}>
      <div role="listbox">{children}</div>
    </SelectContext.Provider>
  );
}

Select.Option = function Option({ value, children }) {
  const { value: selectedValue, onChange } = useContext(SelectContext);
  return (
    <div
      role="option"
      aria-selected={selectedValue === value}
      onClick={() => onChange(value)}
    >
      {children}
    </div>
  );
};

// Usage
function App() {
  const [val, setVal] = useState("b");

  return (
    <Select value={val} onChange={setVal}>
      <Select.Option value="a">Option A</Select.Option>
      <Select.Option value="b">Option B</Select.Option>
      <Select.Option value="c">Option C</Select.Option>
    </Select>
  );
}
```

#### ❓ Which option has `aria-selected="true"` initially? What happens when Option A is clicked?

<details>
<summary>✅ Answer</summary>

```txt
Initially: Option B has aria-selected="true" (value="b" matches selectedValue="b")
After clicking Option A: val becomes "a", Option A gets aria-selected="true",
Option B loses it.
```

**Explanation:** This is the compound components pattern. `Select` owns the state via context. Each `Select.Option` reads from the context to determine if it is selected (`selectedValue === value`). When Option A is clicked, `onChange("a")` is called, which calls `setVal("a")`, triggering a re-render. The context value updates, and all `Select.Option` instances re-evaluate their `aria-selected`. The parent's `val` becomes `"a"`, so only Option A has `aria-selected="true"`.

</details>

---

### Q5

```jsx
function withData(WrappedComponent) {
  return function WithDataComponent({ id, ...rest }) {
    const [data, setData] = useState(null);

    useEffect(() => {
      fetchData(id).then(setData);
    }, [id]);

    return <WrappedComponent data={data} {...rest} />;
  };
}

function UserCard({ data, className }) {
  return <div className={className}>{data?.name}</div>;
}

const UserCardWithData = withData(UserCard);

// Usage
<UserCardWithData id={42} className="card" />
```

#### ❓ What props does `UserCard` receive? What happens if a consumer passes `data` directly?

<details>
<summary>✅ Answer</summary>

```txt
UserCard receives: { data: (fetched data or null), className: "card" }
id is consumed by the HOC and NOT passed to UserCard.

If data is passed directly: <UserCardWithData id={42} data={override} />
The HOC's data overrides the consumer's data because the HOC spreads rest first
then passes data: <WrappedComponent data={data} {...rest} />
Wait — actually data from rest is spread AFTER, so {data: hogData, ...rest} where
rest contains {data: override, className: "card"} → override wins.
```

**Explanation:** `const { id, ...rest } = props` extracts `id` from props. `rest` contains everything else (`className`, and potentially a `data` prop). The `WrappedComponent` call is `<WrappedComponent data={data} {...rest} />`. If `rest` contains a `data` prop, the spread `{...rest}` comes AFTER `data={data}`, so the consumer's `data` prop would override the HOC's fetched `data`. This is a prop naming conflict. Production-quality HOCs prefix injected props with the HOC's concern or use a specific namespace to avoid collisions.

</details>

---

## 2. Data Flow

### Q6

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Parent count: {count}</p>
      <Child onIncrement={() => setCount(c => c + 1)} />
    </div>
  );
}

function Child({ onIncrement }) {
  const [localCount, setLocalCount] = useState(0);

  const handleClick = () => {
    setLocalCount(lc => lc + 1);
    onIncrement();
  };

  return (
    <div>
      <p>Child count: {localCount}</p>
      <button onClick={handleClick}>Increment Both</button>
    </div>
  );
}
```

#### ❓ After clicking the button twice, what do both counts show?

<details>
<summary>✅ Answer</summary>

```txt
Parent count: 2
Child count: 2
```

**Explanation:** Each button click calls `handleClick`, which calls both `setLocalCount` and `onIncrement`. React 18 batches these two state updates (one in `Child`, one in `Parent`) into a single re-render cycle. The child's `localCount` increments from `0→1→2` and the parent's `count` increments from `0→1→2`. Both update together. This demonstrates unidirectional data flow: events flow up (child calls `onIncrement`), data flows down (parent's `count` and child's `localCount` are independent state).

</details>

---

### Q7

```jsx
function GrandParent() {
  const [user, setUser] = useState({ name: "Alice", role: "admin" });

  return <Parent user={user} setUser={setUser} />;
}

function Parent({ user, setUser }) {
  // Parent doesn't use user or setUser — only passes them down
  return <Child user={user} setUser={setUser} />;
}

function Child({ user, setUser }) {
  return (
    <div>
      <p>{user.name}</p>
      <button onClick={() => setUser(u => ({ ...u, name: "Bob" }))}>
        Change Name
      </button>
    </div>
  );
}
```

#### ❓ After clicking the button, which components re-render?

<details>
<summary>✅ Answer</summary>

```txt
GrandParent re-renders (state owner)
Parent re-renders (receives new props — user object has new reference)
Child re-renders (receives new props)
```

**Explanation:** `setUser` triggers a re-render of `GrandParent` (the state owner). `GrandParent` passes the new `user` object to `Parent`. Since `Parent` receives new props, it re-renders. `Parent` passes the new `user` to `Child`, which also re-renders. `Parent` re-renders even though it does not use `user` — this is prop drilling's performance cost. Wrapping `Parent` with `React.memo` would skip its re-render if its props are shallowly equal to the previous props.

</details>

---

### Q8

```jsx
const ThemeContext = createContext("light");

function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext.Provider value={theme}>
      <Page />
      <button onClick={() => setTheme("dark")}>Dark</button>
    </ThemeContext.Provider>
  );
}

function Page() {
  console.log("Page renders");
  return <Widget />;
}

function Widget() {
  const theme = useContext(ThemeContext);
  console.log("Widget renders:", theme);
  return <div className={theme}>Content</div>;
}
```

#### ❓ After clicking Dark, which components log to the console?

<details>
<summary>✅ Answer</summary>

```txt
Page renders
Widget renders: dark
```

**Explanation:** Clicking Dark calls `setTheme("dark")`, which re-renders `App`. `App` re-renders `Page` (child of App), so `Page` re-renders and logs. `Widget` is a child of `Page` AND a context consumer. It re-renders because: (1) `Page` re-renders (parent re-render), and (2) the context value changed. Both reasons cause it to re-render, but it only renders once. `Widget` logs `"dark"` because the new context value is `"dark"`.

</details>

---

### Q9

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <Display count={count} />
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

const Display = React.memo(({ count }) => {
  console.log("Display renders:", count);
  return <p>{count}</p>;
});
```

#### ❓ The button is clicked 3 times. How many times does "Display renders" log?

<details>
<summary>✅ Answer</summary>

```txt
3 times:
Display renders: 1
Display renders: 2
Display renders: 3
```

**Explanation:** `React.memo` prevents re-renders when props are shallowly equal to the previous render. Each button click changes `count` from `0→1`, `1→2`, `2→3`. Each time, the `count` prop passed to `Display` is different from the previous render. `React.memo`'s shallow comparison detects `prevCount !== newCount` and allows the re-render. `memo` would only prevent re-renders if `count` were passed the same value twice in a row (e.g., clicking when already at 0 would not cause a re-render).

</details>

---

### Q10

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const items = [1, 2, 3];  // new array on every render

  return (
    <div>
      <Child items={items} />
      <button onClick={() => setCount(c => c + 1)}>Re-render Parent</button>
    </div>
  );
}

const Child = React.memo(({ items }) => {
  console.log("Child renders");
  return <ul>{items.map(i => <li key={i}>{i}</li>)}</ul>;
});
```

#### ❓ After clicking the button, does `Child` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — Child re-renders every time the button is clicked.
React.memo does not help here.
```

**Explanation:** `const items = [1, 2, 3]` creates a brand new array literal on every render of `Parent`. Even though the contents are identical, the array reference is different. `React.memo` uses `Object.is` for shallow comparison: `Object.is([1,2,3], [1,2,3])` → `false` (different references). So `memo` sees different `items` props every render and allows the re-render. Fix: either define `items` outside the component (if truly static), or use `useMemo`: `const items = useMemo(() => [1, 2, 3], [])`.

</details>

---

## 3. Error Boundaries

### Q11

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) return <p>Error occurred</p>;
    return this.props.children;
  }
}

function BrokenComponent() {
  throw new Error("Render error!");
}

function App() {
  return (
    <ErrorBoundary>
      <BrokenComponent />
    </ErrorBoundary>
  );
}
```

#### ❓ What renders in the browser?

<details>
<summary>✅ Answer</summary>

```txt
"Error occurred"
```

**Explanation:** `BrokenComponent` throws an error during rendering. React propagates this error up the component tree looking for the nearest error boundary. `ErrorBoundary` is the nearest one. React calls `getDerivedStateFromError(error)`, which returns `{ hasError: true }`. The boundary re-renders with `hasError: true` and renders the fallback `<p>Error occurred</p>`. Without the error boundary, the entire app would unmount and show a blank screen (in development, an error overlay appears).

</details>

---

### Q12

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <p>Caught!</p>;
    return this.props.children;
  }
}

function Broken() {
  const [clicked, setClicked] = useState(false);

  if (clicked) throw new Error("Event error");

  return <button onClick={() => setClicked(true)}>Click to break</button>;
}

function App() {
  return (
    <ErrorBoundary>
      <Broken />
    </ErrorBoundary>
  );
}
```

#### ❓ Initially the button renders. After clicking, what happens?

<details>
<summary>✅ Answer</summary>

```txt
After clicking: "Caught!" renders.
The error boundary catches the error.
```

**Explanation:** Clicking the button calls `setClicked(true)`, which triggers a re-render of `Broken`. During this re-render, `if (clicked) throw new Error(...)` executes and throws. This is a render-phase error — it happens during React's render call, not inside a synthetic event handler. Error boundaries catch render-phase errors from child components. The boundary intercepts the throw and renders its fallback. Note: if the error were thrown directly inside an onClick handler (not during a re-render), the error boundary would NOT catch it.

</details>

---

### Q13

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <p>Error</p>;
    return this.props.children;
  }
}

function App() {
  return (
    <ErrorBoundary>
      <button
        onClick={() => {
          throw new Error("Click error");
        }}
      >
        Break
      </button>
    </ErrorBoundary>
  );
}
```

#### ❓ The button is clicked. Does the error boundary catch the error?

<details>
<summary>✅ Answer</summary>

```txt
No — the error boundary does NOT catch this error.
The error propagates as an uncaught JavaScript error (browser error).
```

**Explanation:** Error boundaries only catch errors that occur during rendering, in lifecycle methods, and in constructors. They do NOT catch errors in event handlers. The `onClick` handler runs in response to a browser event, outside React's render cycle. React does not wrap event handlers in try/catch. To handle errors in event handlers, use a regular try/catch block: `onClick={() => { try { throw ... } catch(e) { setState({ error: e }) } }}`.

</details>

---

### Q14

```jsx
class OuterBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <p>Outer caught!</p>;
    return this.props.children;
  }
}

class InnerBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <p>Inner caught!</p>;
    return this.props.children;
  }
}

function Broken() {
  throw new Error("Broken!");
}

function App() {
  return (
    <OuterBoundary>
      <InnerBoundary>
        <Broken />
      </InnerBoundary>
    </OuterBoundary>
  );
}
```

#### ❓ What renders? Which boundary catches the error?

<details>
<summary>✅ Answer</summary>

```txt
"Inner caught!"
InnerBoundary catches the error.
OuterBoundary does not catch it.
```

**Explanation:** React propagates errors upward through the component tree looking for the nearest error boundary. `Broken` throws. React finds `InnerBoundary` as the nearest ancestor with error boundary behavior. `InnerBoundary` catches the error and renders `"Inner caught!"`. `OuterBoundary` never sees the error because `InnerBoundary` already handled it. This is how you create granular error recovery: inner boundaries handle specific feature errors, outer boundaries handle catastrophic fallbacks.

</details>

---

### Q15

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <p>Error!</p>;
    return this.props.children;
  }
}

function App() {
  return (
    <ErrorBoundary>
      <FetchComponent />
    </ErrorBoundary>
  );
}

function FetchComponent() {
  useEffect(() => {
    fetch("/api/data")
      .then(r => r.json())
      .then(data => processData(data))
      .catch(err => {
        throw err;  // throwing inside a promise chain
      });
  }, []);

  return <p>Loading...</p>;
}
```

#### ❓ If the fetch fails, does the error boundary catch the error?

<details>
<summary>✅ Answer</summary>

```txt
No — the error boundary does NOT catch asynchronous errors.
The error is an unhandled promise rejection.
```

**Explanation:** Error boundaries only catch synchronous errors during rendering and lifecycle methods. Errors thrown inside `useEffect` callbacks, promise `.catch()` handlers, or setTimeout are asynchronous — they run outside React's render cycle and are not caught by error boundaries. To surface async errors to an error boundary, you must call a state setter that causes the component to throw during its next render: `setState(() => { throw err; })` or use a library like `react-error-boundary` which provides a `useErrorBoundary()` hook for this purpose.

</details>

---

## 4. Suspense

### Q16

```jsx
const LazyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  const [show, setShow] = useState(false);

  return (
    <div>
      <button onClick={() => setShow(true)}>Load</button>
      {show && (
        <Suspense fallback={<p>Loading...</p>}>
          <LazyComponent />
        </Suspense>
      )}
    </div>
  );
}
```

#### ❓ Initially, what renders? What renders immediately after clicking Load? What renders after the component loads?

<details>
<summary>✅ Answer</summary>

```txt
Initially: just the button (LazyComponent not imported yet)
Immediately after clicking: "Loading..." (Suspense fallback while chunk downloads)
After component loads: the actual LazyComponent renders
```

**Explanation:** `React.lazy` returns a special lazy component that React knows about. When React tries to render `LazyComponent` for the first time, the dynamic `import()` is not yet resolved. React "suspends" — it throws a Promise internally. The nearest `Suspense` boundary catches this suspension and renders its `fallback`. When the import Promise resolves (the chunk downloads), React retries rendering `LazyComponent`, this time successfully, and the fallback is replaced with the actual component.

</details>

---

### Q17

```jsx
const Heavy = lazy(() => import("./Heavy"));

function App() {
  return (
    <Suspense fallback={<p>Loading app...</p>}>
      <div>
        <Header />
        <Suspense fallback={<p>Loading content...</p>}>
          <Heavy />
        </Suspense>
        <Footer />
      </div>
    </Suspense>
  );
}
```

#### ❓ While `Heavy` is loading, what is visible on screen?

<details>
<summary>✅ Answer</summary>

```txt
The Header, Footer, and "Loading content..." are visible.
The outer "Loading app..." fallback is NOT shown.
```

**Explanation:** Suspense boundaries are hierarchical. When `Heavy` suspends, React looks for the nearest Suspense boundary — the inner one. The inner Suspense renders `"Loading content..."` while `Heavy` loads. `Header` and `Footer` are siblings of the inner Suspense, not children of it, so they render normally. The outer Suspense boundary is only activated if something at the top level suspends. Placing Suspense boundaries close to the suspending component keeps the rest of the UI visible.

</details>

---

### Q18

```jsx
const UserData = lazy(() =>
  new Promise(resolve =>
    setTimeout(() => resolve({ default: () => <p>User loaded!</p> }), 2000)
  )
);

function App() {
  const [show, setShow] = useState(false);

  return (
    <div>
      <button onClick={() => setShow(true)}>Show</button>
      <Suspense fallback={<p>Loading...</p>}>
        {show && <UserData />}
      </Suspense>
    </div>
  );
}
```

#### ❓ After clicking Show, describe the render sequence over time.

<details>
<summary>✅ Answer</summary>

```txt
t=0 (click): "Loading..." appears (UserData suspends, Suspense shows fallback)
t=2s (resolves): "User loaded!" appears (Suspense replaces fallback with UserData)
```

**Explanation:** `React.lazy` accepts a function that returns a Promise resolving to `{ default: Component }`. Here a custom 2-second delay simulates a slow chunk download. When `show` becomes `true` and `UserData` first renders, React calls the lazy factory, which returns a pending Promise. React suspends and the nearest `Suspense` shows its fallback. Two seconds later, the Promise resolves with the component. React's internal scheduler detects the resolution and re-renders, replacing the fallback with `<p>User loaded!</p>`.

</details>

---

### Q19

```jsx
function App() {
  const [isPending, startTransition] = useTransition();
  const [tab, setTab] = useState("home");

  const switchTab = (newTab) => {
    startTransition(() => {
      setTab(newTab);
    });
  };

  return (
    <div>
      <button onClick={() => switchTab("profile")}>Profile</button>
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </div>
  );
}
```

#### ❓ When the Profile button is clicked, what renders during the transition?

<details>
<summary>✅ Answer</summary>

```txt
The current tab content (home) remains visible.
The Spinner appears (isPending = true).
When the transition completes, the profile tab content shows and Spinner disappears.
```

**Explanation:** `startTransition` marks the `setTab` update as non-urgent. React defers the re-render of `TabContent` (which might be expensive or trigger a Suspense) and immediately sets `isPending = true`. The UI remains on the "home" tab while React works on the "profile" tab in the background. `isPending` is `true` during this time, showing the Spinner as a hint. When the transition completes, `tab` becomes `"profile"`, `isPending` becomes `false`, `TabContent` renders the new tab, and the Spinner hides. This prevents the UI from freezing on the old tab while keeping it interactive.

</details>

---

### Q20

```jsx
const ProfileData = lazy(() => import("./ProfileData"));

function Profile() {
  return (
    <Suspense fallback={<p>Loading profile...</p>}>
      <ProfileData />
    </Suspense>
  );
}

function App() {
  const [show, setShow] = useState(false);

  return (
    <div>
      <button onClick={() => setShow(true)}>Show Profile</button>
      {show && <Profile />}
    </div>
  );
}
```

#### ❓ The user clicks Show Profile. The `ProfileData` chunk is already cached in the browser from a previous visit. What renders?

<details>
<summary>✅ Answer</summary>

```txt
ProfileData renders immediately (no "Loading profile..." flash).
```

**Explanation:** `React.lazy` with `import()` uses the browser's module cache. If the chunk for `ProfileData` has already been downloaded and parsed in the current browser session, the dynamic `import()` resolves synchronously (from the module registry cache). React does not need to suspend because the lazy component is already available. The Suspense fallback only shows when the Promise is pending. With a cached module, there is no pending Promise — the component renders immediately on the first render attempt.

</details>

---

## 5. Performance Patterns

### Q21

```jsx
const config = { theme: "dark", fontSize: 16 };

const ExpensiveComponent = React.memo(({ config }) => {
  console.log("ExpensiveComponent renders");
  return <div>{config.theme}</div>;
});

function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <ExpensiveComponent config={config} />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </div>
  );
}
```

#### ❓ The button is clicked 5 times. How many times does `ExpensiveComponent` log?

<details>
<summary>✅ Answer</summary>

```txt
1 time — only on the initial render.
```

**Explanation:** `config` is defined outside the component, so it is a module-level constant with a stable reference. On every button click, `App` re-renders, but `config` is the same object reference. `React.memo` performs shallow comparison: `Object.is(prevConfig, newConfig)` → `true` (same reference). No re-render. This illustrates that `React.memo` works well when props are stable references (primitives, refs, functions from `useCallback`, objects from `useMemo`, or module-level constants). The common pitfall is defining objects/arrays inside the component body, creating new references every render.

</details>

---

### Q22

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    console.log("clicked");
  };

  return (
    <div>
      <Child onClick={handleClick} />
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

const Child = React.memo(({ onClick }) => {
  console.log("Child renders");
  return <button onClick={onClick}>Action</button>;
});
```

#### ❓ The `+` button is clicked 3 times. How many times does `Child` re-render?

<details>
<summary>✅ Answer</summary>

```txt
3 times — once per click.
React.memo does not help here.
```

**Explanation:** `handleClick` is defined inside `Parent`. Each render of `Parent` creates a new function instance. Even though `handleClick` has the same code, `Object.is(prevHandleClick, newHandleClick)` is `false` (different function references). `React.memo` sees a changed prop on every render and allows the re-render. Fix: wrap `handleClick` in `useCallback(() => { console.log("clicked"); }, [])`. This stabilizes the function reference and `memo`'s comparison succeeds — `Child` only renders once.

</details>

---

### Q23

```jsx
function App() {
  const [value, setValue] = useState("");

  const processedItems = useMemo(() => {
    console.log("Computing items");
    return Array.from({ length: 1000 }, (_, i) => ({ id: i, label: `Item ${i}` }));
  }, []);

  return (
    <div>
      <input value={value} onChange={e => setValue(e.target.value)} />
      <ul>{processedItems.map(item => <li key={item.id}>{item.label}</li>)}</ul>
    </div>
  );
}
```

#### ❓ The user types 10 characters. How many times is "Computing items" logged?

<details>
<summary>✅ Answer</summary>

```txt
1 time — only on the initial render.
```

**Explanation:** `useMemo(() => ..., [])` runs the computation only once because the dependency array is empty. On every subsequent render (each keystroke), `useMemo` returns the cached result without running the computation. The 1000-item array is computed once and memoized. Typing triggers `setValue` which re-renders `App`, but `processedItems` is served from the memo cache. The list renders without recomputation. This is the correct use of `useMemo` — expensive computation whose result does not depend on the changing state.

</details>

---

### Q24

```jsx
function ProductList({ products }) {
  const handleProductClick = useCallback((product) => {
    console.log("clicked:", product.name);
    trackAnalytics(product);
  }, []);  // empty deps

  return (
    <ul>
      {products.map(product => (
        <ProductItem
          key={product.id}
          product={product}
          onClick={handleProductClick}
        />
      ))}
    </ul>
  );
}
```

#### ❓ Is the empty `[]` dependency array correct? What would happen if `products` changed and a product handler needed the latest `products` list?

<details>
<summary>✅ Answer</summary>

```txt
For this specific code: yes, [] is correct because the callback
does not reference any component state or props.

If the handler needed products: [] would create a stale closure.
The callback would always reference the initial products array.
Fix: add products to the dependency array or use useRef for the latest value.
```

**Explanation:** `useCallback` with `[]` creates the function once and memoizes it forever. The function's closure captures whatever variables are in scope at creation time. Here, `trackAnalytics` is presumably a module-level import — stable. But if the handler were `(product) => console.log(products.indexOf(product))`, it would close over the initial `products` array and would always reference stale data. The rule: any state or prop variable used inside a `useCallback` must be in the dependency array (or the ESLint exhaustive-deps rule will warn you).

</details>

---

### Q25

```jsx
const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  const [Component, setComponent] = useState(null);

  const loadComponent = () => {
    setComponent(() => HeavyComponent);
  };

  return (
    <div>
      <button onClick={loadComponent}>Load</button>
      {Component && (
        <Suspense fallback={<p>Loading...</p>}>
          <Component />
        </Suspense>
      )}
    </div>
  );
}
```

#### ❓ Is storing a lazy component in `useState` correct? What is the common mistake here?

<details>
<summary>✅ Answer</summary>

```txt
There is a subtle bug: setComponent(() => HeavyComponent) uses the functional
updater form — React treats the function as a state initializer, so Component
receives the return value of calling HeavyComponent(), not HeavyComponent itself.

Correct: setComponent(HeavyComponent) — passes the component directly as the value.
```

**Explanation:** `useState`'s setter accepts either a value or a function. If you pass a function, React calls it as `fn(prevState)` to compute the new state. `setComponent(() => HeavyComponent)` passes an arrow function `() => HeavyComponent`. React calls this function and stores its return value — which is `HeavyComponent` the component. So it actually works in this case because the arrow function returns `HeavyComponent`! But `setComponent(HeavyComponent)` would NOT work — React would treat `HeavyComponent` (a function component) as a state updater and call it, storing `HeavyComponent(undefined)` (the rendered output) as state. For storing functions in state, always use the `() => fn` wrapper form.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Component Patterns | Q1–Q5 | HOC displayName, render props, compound components, HOC prop conflict |
| Data Flow | Q6–Q10 | Events up / data down, re-render propagation, context re-renders, React.memo with unstable references |
| Error Boundaries | Q11–Q15 | Render error caught, state-triggered render error, event handler not caught, nested boundaries, async not caught |
| Suspense | Q16–Q20 | Lazy component loading sequence, nested Suspense, transition pending, cached module instant render |
| Performance Patterns | Q21–Q25 | Stable reference with memo, useCallback necessity, useMemo with empty deps, stale closure in useCallback, function state storage |
