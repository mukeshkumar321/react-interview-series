# React Performance

## Table of Contents

- [1. Why React Renders](#1-why-react-renders)
- [2. React.memo](#2-reactmemo)
- [3. useMemo](#3-usememo)
- [4. useCallback](#4-usecallback)
- [5. React Profiler and DevTools](#5-react-profiler-and-devtools)
- [6. Code Splitting](#6-code-splitting)
- [7. Virtualization and Windowing](#7-virtualization-and-windowing)
- [8. Image Optimization](#8-image-optimization)
- [9. Bundle Size Optimization](#9-bundle-size-optimization)
- [10. Common Performance Anti-Patterns](#10-common-performance-anti-patterns)
- [11. Concurrent Features in React 18](#11-concurrent-features-in-react-18)
- [12. Web Vitals and Core Web Vitals](#12-web-vitals-and-core-web-vitals)
- [13. Performance Measurement Techniques](#13-performance-measurement-techniques)
- [14. Best Practices](#14-best-practices)

---

## 1. Why React Renders

Understanding when React re-renders is the foundation of all performance work.

---

### The Four Causes of Re-render

React re-renders a component when any of these four things happen:

1. **State changes** — `setState` is called with a new value
2. **Props change** — the parent passes new prop values
3. **Parent re-renders** — when a parent re-renders, all children re-render by default
4. **Context value changes** — any component consuming a changed context re-renders

```text
Parent re-renders
    ↓
All children re-render
    ↓
All grandchildren re-render
    ↓ (and so on down the tree)
```

---

### What Re-rendering Actually Means

Re-rendering means React calls your component function again to produce a new Virtual DOM description. It does NOT mean the actual browser DOM updates — React diffs the old and new Virtual DOM and only updates what changed.

```jsx
function Counter({ label }) {
  const [count, setCount] = useState(0);
  console.log("Counter rendered"); // called on every render

  return (
    <div>
      <p>{label}: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}
```

Re-renders are not inherently expensive unless the component function is slow or the resulting DOM diff produces large changes.

---

### When Re-rendering is a Problem

Re-rendering becomes a problem when:
- A parent renders frequently and has many children
- A child component runs expensive computations during render
- A context value changes and many components consume it
- Lists re-render all items when only one item changed

---

### Object.is Comparison for Bailout

React uses `Object.is` to compare old and new state/props. If they are equal, React bails out (skips the re-render).

```jsx
// Primitive — bailout works
setState(0); // Object.is(0, 0) = true → no re-render

// Object — no bailout (new reference every time)
setState({ count: 0 }); // Object.is(old, new) = false → re-render
```

---

## 2. React.memo

`React.memo` is a higher-order component (HOC) that wraps a component and memoizes its render output. It only re-renders if props change.

---

### How React.memo Works

```jsx
const MemoizedComponent = React.memo(MyComponent);
```

Before re-rendering `MemoizedComponent`, React shallowly compares old props with new props. If all props are equal (via `Object.is`), the previous render output is reused.

---

### Basic Usage

```jsx
const UserCard = React.memo(function UserCard({ user }) {
  console.log("UserCard rendered");
  return <div>{user.name} — {user.email}</div>;
});

function App() {
  const [count, setCount] = useState(0);
  const user = { name: "Alice", email: "alice@example.com" };

  // App re-renders on every count change, but UserCard is memoized
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <UserCard user={user} /> {/* re-renders every time — object is new */}
    </>
  );
}
```

**Problem above:** `user` is a new object on every render. `Object.is(oldUser, newUser)` is `false` even though the data is identical. `React.memo` does not help here.

---

### Shallow Comparison

React.memo performs a **shallow comparison** of props. This means:
- Primitive values (`string`, `number`, `boolean`) are compared by value
- Objects and arrays are compared by reference

```jsx
// Shallow comparison
React.memo(Component);

// Equivalent manual check:
const arePropsEqual = (prevProps, nextProps) => {
  return Object.keys(prevProps).every(key =>
    Object.is(prevProps[key], nextProps[key])
  );
};
```

---

### When React.memo Prevents Re-render

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoizedChild name="Alice" /> {/* stable string prop — skips re-render */}
    </>
  );
}

const MemoizedChild = React.memo(function Child({ name }) {
  console.log("Child rendered"); // only logs on mount
  return <p>{name}</p>;
});
```

---

### Custom Comparison Function

Pass a second argument to `React.memo` for custom comparison logic:

```jsx
const UserCard = React.memo(
  function UserCard({ user }) {
    return <div>{user.name}</div>;
  },
  (prevProps, nextProps) => {
    // Return true to skip re-render, false to re-render
    return prevProps.user.id === nextProps.user.id &&
           prevProps.user.name === nextProps.user.name;
  }
);
```

The second argument receives previous and next props. Return `true` to skip re-render (props are "equal"), `false` to re-render.

---

### When React.memo is Not Needed

- Components that re-render because their own state changed (memo cannot prevent this)
- Lightweight components where the memo comparison cost exceeds the render cost
- Components that always receive new props anyway

---

## 3. useMemo

`useMemo` memoizes the result of an expensive computation. It recomputes only when its dependencies change.

---

### Syntax

```jsx
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

- The first argument is a function that returns the computed value
- The second argument is the dependency array
- The function runs on mount and whenever a dependency changes

---

### Preventing Expensive Recomputation

```jsx
function ProductList({ products, filter }) {
  // Runs on every render — expensive for large lists
  const filtered = products.filter(p =>
    p.name.toLowerCase().includes(filter.toLowerCase())
  );

  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

```jsx
function ProductList({ products, filter }) {
  // Only recomputes when products or filter changes
  const filtered = useMemo(
    () => products.filter(p =>
      p.name.toLowerCase().includes(filter.toLowerCase())
    ),
    [products, filter]
  );

  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

---

### Referential Equality for Downstream Components

`useMemo` is also used to stabilize object/array references for child components wrapped in `React.memo`:

```jsx
function Parent({ userId }) {
  // Without useMemo: new object reference every render → child always re-renders
  const config = useMemo(() => ({ userId, theme: 'dark' }), [userId]);

  return <MemoizedChild config={config} />;
}
```

---

### useMemo vs Variable Declaration

```jsx
// Without useMemo — recomputes on every render
const sortedList = [...items].sort(compareFn);

// With useMemo — only recomputes when items changes
const sortedList = useMemo(() => [...items].sort(compareFn), [items]);
```

Use `useMemo` when the computation is measurably expensive. For cheap operations, the overhead of `useMemo` itself (dependency comparison + closure) may exceed the savings.

---

### When Not to Use useMemo

❌ Wrong — useMemo on a trivial computation

```jsx
// Overhead of useMemo > cost of computation
const doubled = useMemo(() => count * 2, [count]);

// Better — just compute it
const doubled = count * 2;
```

---

## 4. useCallback

`useCallback` memoizes a function reference. It returns the same function instance between renders unless its dependencies change.

---

### Syntax

```jsx
const memoizedCallback = useCallback(
  () => doSomething(a, b),
  [a, b]
);
```

---

### Why Function References Matter

Without `useCallback`, a new function is created on every render:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  // New function every render
  const handleClick = () => console.log(count);

  return <MemoizedChild onClick={handleClick} />;
}
```

Even though `MemoizedChild` is wrapped in `React.memo`, it re-renders on every `Parent` render because `handleClick` is a new reference each time.

---

### useCallback with React.memo

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState("");

  // Stable reference — only recreates when count changes
  const handleCount = useCallback(() => {
    setCount(c => c + 1);
  }, []); // empty deps — functional update avoids stale closure

  return (
    <>
      <input value={name} onChange={e => setName(e.target.value)} />
      <MemoizedButton onClick={handleCount} label="Increment" />
      {/* MemoizedButton does NOT re-render when name changes */}
    </>
  );
}

const MemoizedButton = React.memo(function Button({ onClick, label }) {
  console.log("Button rendered");
  return <button onClick={onClick}>{label}</button>;
});
```

---

### useCallback for Dependencies

`useCallback` is also used when a function is a dependency of `useEffect`, `useMemo`, or another `useCallback`:

```jsx
const fetchData = useCallback(async () => {
  const data = await api.get(`/users/${userId}`);
  setData(data);
}, [userId]);

useEffect(() => {
  fetchData();
}, [fetchData]); // fetchData is stable unless userId changes
```

Without `useCallback`, `fetchData` would be a new function every render, making `useEffect`'s dependency unstable and causing an infinite refetch loop.

---

### The Trio: React.memo + useMemo + useCallback

These three work together:

```text
React.memo      → prevents child re-render when props haven't changed
useCallback     → stabilizes function props passed to memoized children
useMemo         → stabilizes object/array props passed to memoized children
```

Using `React.memo` without `useCallback`/`useMemo` for function/object props has no effect.

---

## 5. React Profiler and DevTools

---

### React DevTools Profiler

The React DevTools browser extension includes a Profiler tab that records render timings.

**How to use:**
1. Open DevTools → React tab → Profiler
2. Click record
3. Interact with your app
4. Stop recording
5. Inspect the flame chart

**What to look for:**
- Components with long render times (wide bars in the flame chart)
- Components that re-render unexpectedly (shown in yellow/orange)
- "Why did this render?" panel showing which prop or state changed

---

### React Profiler API

Use the `Profiler` component to measure render times programmatically:

```jsx
import { Profiler } from 'react';

function onRenderCallback(
  id,           // "id" prop of the Profiler
  phase,        // "mount" or "update"
  actualDuration,  // time spent rendering this update
  baseDuration,    // estimated time for full render without memoization
  startTime,    // when React began rendering this update
  commitTime    // when React committed this update
) {
  console.log(`${id} ${phase}: ${actualDuration.toFixed(2)}ms`);
}

function App() {
  return (
    <Profiler id="Navigation" onRender={onRenderCallback}>
      <Navigation />
    </Profiler>
  );
}
```

---

### Identifying Bottlenecks

Common signs of performance issues:

| Symptom | Likely Cause |
|---|---|
| Parent re-render causes all children to re-render | Missing React.memo |
| Function child prop changes every render | Missing useCallback |
| Object child prop changes every render | Missing useMemo |
| Expensive computation runs on every render | Missing useMemo |
| Long task blocking main thread | Missing code splitting or virtualization |

---

## 6. Code Splitting

Code splitting breaks a JavaScript bundle into smaller chunks that are loaded on demand. This reduces the initial bundle size and speeds up page load.

---

### React.lazy and Suspense

`React.lazy` dynamically imports a component. `Suspense` shows a fallback while the chunk loads.

```jsx
import { lazy, Suspense } from 'react';

// Not loaded until the component is rendered
const Dashboard = lazy(() => import('./Dashboard'));
const Settings = lazy(() => import('./Settings'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

---

### Route-Based Code Splitting

The most impactful code splitting strategy is per route — each page loads its own chunk:

```jsx
const Home = lazy(() => import('./pages/Home'));
const Profile = lazy(() => import('./pages/Profile'));
const AdminPanel = lazy(() => import('./pages/AdminPanel'));

// Webpack/Vite creates separate chunks:
// home.[hash].js, profile.[hash].js, adminpanel.[hash].js
// The user downloads only the chunk for their current route
```

---

### Component-Level Splitting

Split heavy components that are not needed on initial render:

```jsx
// Heavy chart library only loads when ChartView renders
const ChartView = lazy(() => import('./ChartView'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<Skeleton />}>
          <ChartView />
        </Suspense>
      )}
    </>
  );
}
```

---

### Named Export with lazy

`React.lazy` only works with default exports. For named exports, create an intermediate module or use an inline re-export:

```jsx
// For a named export Component from './module'
const Component = lazy(() =>
  import('./module').then(mod => ({ default: mod.Component }))
);
```

---

### Preloading

Preload a chunk before the user navigates to it:

```jsx
// Preload on hover — chunk downloads before click
<Link
  to="/dashboard"
  onMouseEnter={() => import('./Dashboard')}
>
  Dashboard
</Link>
```

---

## 7. Virtualization and Windowing

Virtualization renders only the items currently visible in the viewport, regardless of list length.

---

### The Problem

Rendering 10,000 items creates 10,000 DOM nodes:
- Initial render takes seconds
- Memory usage is high
- Scrolling is janky
- Any parent re-render forces reconciliation of all 10,000 items

---

### react-window

`react-window` is a lightweight virtualization library:

```jsx
import { FixedSizeList } from 'react-window';

function Row({ index, style }) {
  return (
    <div style={style}>
      Item {index}
    </div>
  );
}

function VirtualList({ items }) {
  return (
    <FixedSizeList
      height={600}        // container height
      itemCount={items.length}
      itemSize={50}       // each row height
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

Only ~15 DOM nodes are in the page at any time, regardless of list size.

---

### TanStack Virtual (react-virtual)

More flexible than react-window, supports variable row heights:

```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

function VirtualList({ items }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 70,
    overscan: 5, // extra items to render above/below viewport
  });

  return (
    <div ref={parentRef} style={{ height: '500px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualItem.size}px`,
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {items[virtualItem.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

### When to Virtualize

Use virtualization when:
- Lists exceed ~100 items
- Items are rendered in a scrollable container
- Each item has non-trivial rendering cost

---

## 8. Image Optimization

---

### Lazy Loading Images

Images below the fold should not load until the user scrolls to them:

```jsx
// Native lazy loading (modern browsers)
<img src="/photo.jpg" alt="Photo" loading="lazy" />

// With Intersection Observer (custom hook)
function LazyImage({ src, alt }) {
  const imgRef = useRef(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setIsVisible(true);
        observer.disconnect();
      }
    });
    observer.observe(imgRef.current);
    return () => observer.disconnect();
  }, []);

  return (
    <div ref={imgRef}>
      {isVisible ? <img src={src} alt={alt} /> : <div className="placeholder" />}
    </div>
  );
}
```

---

### Always Specify Dimensions

Images without dimensions cause Cumulative Layout Shift (CLS):

```jsx
// Wrong — causes layout shift as image loads
<img src="/hero.jpg" alt="Hero" />

// Correct — browser reserves space before image loads
<img src="/hero.jpg" alt="Hero" width={1200} height={600} />
```

---

### Next.js Image Component

In Next.js, the `<Image>` component handles optimization automatically:

```jsx
import Image from 'next/image';

// Automatic: lazy loading, responsive sizes, WebP conversion, blur placeholder
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority // load eagerly for above-fold images
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
/>
```

---

## 9. Bundle Size Optimization

---

### Tree Shaking

Tree shaking eliminates unused exports from the final bundle. It requires ES module syntax (`import`/`export`):

```jsx
// Wrong — imports the entire lodash library (~70KB)
import _ from 'lodash';
const result = _.debounce(fn, 300);

// Correct — imports only debounce function (~2KB)
import debounce from 'lodash/debounce';
// or
import { debounce } from 'lodash-es'; // ES module version
```

---

### Dynamic Imports for Rarely Used Features

```jsx
// Heavy PDF library only downloads when the user clicks "Export"
async function handleExport() {
  const { generatePDF } = await import('./pdf-generator');
  generatePDF(data);
}
```

---

### Bundle Analysis

Analyze bundle composition to find large dependencies:

```bash
# Vite
npx vite-bundle-visualizer

# Create React App / Webpack
npx webpack-bundle-analyzer build/static/js/*.js
```

Common findings:
- Unused locales in date libraries (moment.js)
- Multiple versions of the same library
- Development-only code in production bundle

---

### Replacing Heavy Libraries

| Heavy Library | Lighter Alternative |
|---|---|
| moment.js (70KB) | date-fns (tree-shakeable) or dayjs (2KB) |
| lodash (70KB) | lodash-es (tree-shakeable) or native JS |
| axios (13KB) | native fetch API |
| Full icon library | Import individual icons |

---

## 10. Common Performance Anti-Patterns

---

### Creating Objects in Render

❌ Wrong

```jsx
function App() {
  return (
    <UserCard
      style={{ color: 'red', margin: 8 }} // new object every render
      config={{ theme: 'dark' }}           // new object every render
    />
  );
}
```

✅ Correct

```jsx
const cardStyle = { color: 'red', margin: 8 }; // defined outside component
const cardConfig = { theme: 'dark' };

function App() {
  return <UserCard style={cardStyle} config={cardConfig} />;
}
```

---

### Inline Function Definitions as Event Handlers

❌ Wrong (when passed to memoized components)

```jsx
function Parent() {
  return (
    <MemoizedButton
      onClick={() => handleAction()} // new function every render
    />
  );
}
```

✅ Correct

```jsx
function Parent() {
  const handleClick = useCallback(() => handleAction(), []);

  return <MemoizedButton onClick={handleClick} />;
}
```

Inline functions for non-memoized components are fine and often preferable for readability.

---

### Using Math.random() or Index as Keys

❌ Wrong

```jsx
{items.map((item, index) => (
  <Item key={Math.random()} item={item} /> // new key every render → full remount
))}

{items.map((item, index) => (
  <Item key={index} item={item} /> // wrong when list can reorder or filter
))}
```

✅ Correct

```jsx
{items.map(item => (
  <Item key={item.id} item={item} /> // stable, unique identity
))}
```

---

### Deep Component Trees Without Memoization

A deeply nested tree where the root frequently re-renders will cascade re-renders through every level. Solutions:

1. Lift state down — keep state as close to where it is used as possible
2. Wrap stable subtrees in `React.memo`
3. Use composition (children prop) to avoid passing props through many levels

---

### Expensive Operations in Render Body

❌ Wrong

```jsx
function App({ data }) {
  // Runs on every render
  const processed = data
    .filter(item => item.active)
    .map(item => transform(item))
    .sort(compareBy('name'));

  return <List items={processed} />;
}
```

✅ Correct

```jsx
function App({ data }) {
  const processed = useMemo(
    () => data
      .filter(item => item.active)
      .map(item => transform(item))
      .sort(compareBy('name')),
    [data]
  );

  return <List items={processed} />;
}
```

---

## 11. Concurrent Features in React 18

React 18 introduced concurrent rendering — the ability to pause, resume, and prioritize renders.

---

### startTransition

Mark a state update as non-urgent so React can interrupt it to handle more urgent updates (like user input):

```jsx
import { startTransition, useState } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  function handleChange(e) {
    const value = e.target.value;

    // Urgent — update input immediately
    setQuery(value);

    // Non-urgent — defer the expensive list update
    startTransition(() => {
      setResults(filterResults(value)); // heavy computation
    });
  }

  return (
    <>
      <input value={query} onChange={handleChange} />
      <ResultsList results={results} />
    </>
  );
}
```

The input stays responsive while `ResultsList` renders in the background.

---

### useTransition

`useTransition` is the hook version of `startTransition`. It also provides an `isPending` flag:

```jsx
import { useTransition, useState } from 'react';

function TabPanel() {
  const [tab, setTab] = useState('posts');
  const [isPending, startTransition] = useTransition();

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <>
      <TabBar onSelect={selectTab} />
      {isPending && <Spinner />}
      <TabContent tab={tab} />
    </>
  );
}
```

`isPending` is `true` while the transition is in progress, allowing you to show a loading indicator without blocking the current tab.

---

### useDeferredValue

Defer updating a value until the browser is idle. Useful for deferring a value derived from a fast-changing input:

```jsx
import { useDeferredValue, useState } from 'react';

function Search() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {/* SlowList renders with deferredQuery — may lag behind input */}
      <SlowList query={deferredQuery} />
    </>
  );
}
```

The input updates immediately. `deferredQuery` updates when React has spare capacity, so `SlowList` re-renders are de-prioritized.

---

### Difference: startTransition vs useDeferredValue

| | startTransition | useDeferredValue |
|---|---|---|
| What you control | A state update | A derived value |
| When to use | You own the state setter | You receive a prop or value from outside |
| isPending available | Yes (via useTransition) | No — use `value !== deferredValue` |

---

### Suspense for Data (React 18+)

React 18 extends Suspense to handle asynchronous data. When a component "suspends" (throws a Promise), the nearest `Suspense` boundary shows its fallback:

```jsx
function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <UserProfile /> {/* may suspend while fetching */}
    </Suspense>
  );
}
```

Libraries like React Query and Relay support this via `suspense: true` option.

---

## 12. Web Vitals and Core Web Vitals

Core Web Vitals are Google's metrics for user experience quality. They directly affect SEO ranking.

---

### LCP — Largest Contentful Paint

Measures how long the largest visible element takes to render.

- Good: < 2.5 seconds
- Needs improvement: 2.5s – 4s
- Poor: > 4s

**Improvements:**
- Preload the hero image: `<link rel="preload" as="image" href="/hero.jpg">`
- Use `priority` on the above-fold image in Next.js
- Reduce server response time
- Eliminate render-blocking CSS/JS

---

### INP — Interaction to Next Paint (replaced FID in 2024)

Measures the latency of all interactions (click, tap, keyboard) during the page lifetime.

- Good: < 200ms
- Needs improvement: 200ms – 500ms
- Poor: > 500ms

**Improvements:**
- Avoid long tasks on the main thread (> 50ms)
- Use `startTransition` for non-urgent state updates
- Defer non-critical JavaScript with `requestIdleCallback`
- Break long tasks with `scheduler.yield()`

---

### CLS — Cumulative Layout Shift

Measures visual stability — how much page content moves unexpectedly.

- Good: < 0.1
- Needs improvement: 0.1 – 0.25
- Poor: > 0.25

**Improvements:**
- Always specify `width` and `height` on images
- Reserve space for dynamic content (ads, embeds) with CSS aspect-ratio
- Avoid inserting content above existing content
- Use CSS transforms instead of top/left for animations

---

### Web Vitals Summary

| Metric | Measures | Good Threshold |
|---|---|---|
| LCP | Largest visible element paint time | < 2.5s |
| INP | Interaction responsiveness | < 200ms |
| CLS | Layout stability | < 0.1 |
| TTFB | Time to First Byte (server) | < 800ms |
| FCP | First Contentful Paint | < 1.8s |

---

## 13. Performance Measurement Techniques

---

### React DevTools Profiler

Record renders, inspect why components re-rendered, compare before/after memoization.

---

### Performance.mark and Performance.measure

```jsx
function measureComponent() {
  performance.mark('render-start');
  // ... render
  performance.mark('render-end');
  performance.measure('render', 'render-start', 'render-end');
  console.log(performance.getEntriesByName('render')[0].duration);
}
```

---

### web-vitals Library

```jsx
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(metric => console.log('LCP:', metric.value));
onINP(metric => console.log('INP:', metric.value));
onCLS(metric => console.log('CLS:', metric.value));
```

---

### Lighthouse

Run in Chrome DevTools → Lighthouse tab. Generates a performance score with specific actionable recommendations.

---

### Why Did You Render

`@welldone-software/why-did-you-render` patches React to log extra re-render information in development:

```jsx
import React from 'react';
import whyDidYouRender from '@welldone-software/why-did-you-render';

if (process.env.NODE_ENV === 'development') {
  whyDidYouRender(React, {
    trackAllPureComponents: true,
  });
}
```

---

## 14. Best Practices

---

### Performance Optimization Order

1. **Profile first** — do not optimize blindly. Measure where the actual bottleneck is.
2. **Fix the biggest wins first** — code splitting, virtualization, and removing heavy dependencies often have much more impact than individual component memoization.
3. **Apply memoization surgically** — only memoize when you have measured a problem.

---

### Quick Reference

| Problem | Solution |
|---|---|
| Parent re-renders cascade to all children | React.memo |
| Expensive computation runs every render | useMemo |
| Function prop reference changes every render | useCallback |
| Large initial bundle | Code splitting with React.lazy |
| Long list performance | Virtualization with react-window |
| UI jank on slow state update | startTransition / useTransition |
| Input lags while expensive list updates | useDeferredValue |
| Images cause layout shift | width + height attributes |
| Heavy library inflating bundle | Tree shaking or lighter alternative |
| N+1 renders from context | Split context into smaller providers |

---

### Memoization Is Not Free

Every `useMemo`, `useCallback`, and `React.memo` has a cost:
- Memory for storing the cached value
- Time for dependency comparison on every render
- Complexity cost for future readers

Optimize reactively, not proactively. Write clear code first, then profile and optimize specific bottlenecks.

---

### Context Performance

Context is re-evaluated on every render of the Provider. All consumers re-render when the context value changes.

```jsx
// Wrong — new object on every render triggers all consumers to re-render
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
}

// Better — split into separate contexts for reads vs writes
const AuthStateContext = createContext(null);
const AuthDispatchContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return (
    <AuthStateContext.Provider value={user}>
      <AuthDispatchContext.Provider value={setUser}>
        {children}
      </AuthDispatchContext.Provider>
    </AuthStateContext.Provider>
  );
}
```

Components that only call `setUser` (login/logout buttons) subscribe to `AuthDispatchContext`. `setUser` is stable (same reference), so they never re-render when `user` changes.
