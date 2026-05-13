## useMemo — Tricky Output Questions

> These questions test your understanding of when `useMemo` recomputes, dependency array behavior, referential equality, interaction with `React.memo`, and edge cases. Each question reflects a real scenario from senior React interviews.

---

## 1. Basic Memoization

### Q1

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [other, setOther] = useState(0);

  const expensive = useMemo(() => {
    console.log('computing...');
    return count * 2;
  }, [count]);

  return (
    <>
      <p>{expensive}</p>
      <button onClick={() => setCount(c => c + 1)}>Count +</button>
      <button onClick={() => setOther(o => o + 1)}>Other +</button>
    </>
  );
}
```

#### ❓ "Other +" is clicked 5 times, then "Count +" is clicked once. How many times is "computing..." logged total?

<details>
<summary>✅ Answer</summary>

```txt
2 times total:
1. On initial render
2. After "Count +" is clicked
```

**Explanation:** `useMemo` with `[count]` only recomputes when `count` changes. Clicking "Other +" changes `other`, which causes a re-render, but `count` is unchanged — so `useMemo` returns the cached value and "computing..." is NOT logged. Clicking "Count +" changes `count`, triggering recomputation. Total: initial render (1) + one count click (1) = 2 logs.

</details>

---

### Q2

```jsx
function App() {
  const [a, setA] = useState(1);
  const [b, setB] = useState(2);

  const sum = useMemo(() => {
    console.log('sum computed');
    return a + b;
  }, [a, b]);

  return (
    <>
      <p>{sum}</p>
      <button onClick={() => { setA(2); setB(3); }}>Update Both</button>
    </>
  );
}
```

#### ❓ After clicking "Update Both", how many times is "sum computed" logged?

<details>
<summary>✅ Answer</summary>

```txt
1 time (after the button click, totaling 2: initial + click)
```

**Explanation:** React 18 batches both `setA(2)` and `setB(3)` into a single re-render. After that single re-render, `useMemo` checks its deps. Both `a` and `b` changed, so it recomputes — but only once, because there was only one render. "sum computed" is logged once during the initial render and once after the button click. Total: 2 logs.

</details>

---

### Q3

```jsx
function App() {
  const [count, setCount] = useState(0);

  const result = useMemo(() => {
    console.log('useMemo ran');
    return count;
  }, [count]);

  console.log('rendered');

  return <button onClick={() => setCount(0)}>Set to 0</button>;
}
```

#### ❓ After the initial render, clicking the button 3 times — what is logged?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
useMemo ran
rendered

// After each click (3 times — each click calls setCount(0) when count is already 0):
rendered  ← React may run the component once to confirm bailout
```

**Explanation:** `setCount(0)` when `count` is already `0` uses `Object.is(0, 0) = true` — React bails out. However, React may still call the component function once to confirm the bailout. After the bailout confirmation, `useMemo` sees `count` is still `0` and returns the cached value — "useMemo ran" is NOT logged again. Only "rendered" may appear (once, for the bailout check). After the first bailout confirmation, subsequent same-value calls skip even the component function.

</details>

---

### Q4

```jsx
function App() {
  const [count, setCount] = useState(0);

  const value = useMemo(() => {
    return { count, doubled: count * 2 };
  }, [count]);

  console.log('render, value ref:', value);

  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

#### ❓ After 3 button clicks, are the 4 logged `value` references (initial + 3 clicks) the same object?

<details>
<summary>✅ Answer</summary>

```txt
No — each time count changes, useMemo creates a new object.
All 4 logged values are different object references.
```

**Explanation:** `useMemo` recomputes when `count` changes. Each recomputation creates and returns a new `{ count, doubled }` object. So each of the 4 renders (initial + 3 clicks) produces a distinct object reference. `useMemo` caches the result of the computation — and the computation here returns a new object each time. The cache is only reused between renders where deps are unchanged.

</details>

---

### Q5

```jsx
function App() {
  const [x, setX] = useState(5);

  const result = useMemo(() => {
    console.log('memo ran');
    return x * x;
  }, [x]);

  return (
    <>
      <p>{result}</p>
      <button onClick={() => setX(5)}>Set to 5 again</button>
    </>
  );
}
```

#### ❓ In React 18 Strict Mode (development), how many times does "memo ran" log on initial render? Why?

<details>
<summary>✅ Answer</summary>

```txt
2 times in Strict Mode development.
```

**Explanation:** React 18 Strict Mode intentionally double-invokes the function passed to `useMemo` during development to verify it is pure (no side effects). If the function is pure, calling it twice should produce the same result both times. This is why `console.log('memo ran')` appears twice on mount in development. In production, the function is called exactly once. This is the same behavior as component render functions and state initializers.

</details>

---

## 2. Dependency Array

### Q6

```jsx
function App() {
  const [count, setCount] = useState(0);

  const obj = { value: count }; // new object every render

  const result = useMemo(() => {
    console.log('computing');
    return obj.value * 10;
  }, [obj]); // obj is the dependency

  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

#### ❓ The button is clicked 5 times. How many times does "computing" log?

<details>
<summary>✅ Answer</summary>

```txt
6 times (initial render + 5 clicks)
```

**Explanation:** `obj` is created inline as a new object on every render. `Object.is({...}, {...})` compares by reference — two different objects are never equal even if they have the same shape. So every render produces a new `obj` reference, `useMemo` sees a "changed" dep, and recomputes every time. This completely defeats memoization. Fix: depend on `count` directly — `useMemo(() => count * 10, [count])`.

</details>

---

### Q7

```jsx
function App({ config }) {
  const [text, setText] = useState('');

  const processed = useMemo(() => {
    console.log('processing');
    return processData(text, config.threshold);
  }, [text]); // config is missing from deps

  return <input value={text} onChange={e => setText(e.target.value)} />;
}
```

#### ❓ If `config.threshold` changes (parent passes new config), does `processed` update?

<details>
<summary>✅ Answer</summary>

```txt
No — processed does NOT update when config.threshold changes.
```

**Explanation:** `config` is missing from the dependency array. `useMemo` only recomputes when `text` changes. If the parent passes a new `config` with a different `threshold`, the component re-renders but `useMemo` sees `text` is unchanged and returns the stale cached result — computed with the old `config.threshold`. This is a missing dependency bug. The ESLint `exhaustive-deps` rule would flag this. Fix: `[text, config]` or `[text, config.threshold]`.

</details>

---

### Q8

```jsx
function App() {
  const [count, setCount] = useState(3);

  const doubled = useMemo(() => {
    console.log('computed');
    return count * 2;
  }, [count]);

  return (
    <>
      <p>{doubled}</p>
      <button onClick={() => setCount(3)}>Set to 3</button>
    </>
  );
}
```

#### ❓ After the initial render, "Set to 3" is clicked. Does `useMemo` recompute?

<details>
<summary>✅ Answer</summary>

```txt
No — useMemo does NOT recompute. "computed" is not logged again.
```

**Explanation:** Clicking "Set to 3" when `count` is already `3` causes a state bailout: `Object.is(3, 3) = true`. React bails out and skips re-rendering entirely (or at most runs the component once to confirm the bailout without DOM update). Since there is no meaningful re-render, `useMemo` never even gets to compare deps. "computed" is only logged on the initial render.

</details>

---

### Q9

```jsx
function App() {
  const [count, setCount] = useState(0);

  const fn = () => count * 2; // new function every render

  const result = useMemo(() => {
    console.log('running fn');
    return fn();
  }, [fn]); // fn is a dep

  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

#### ❓ How often does "running fn" log when the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
Every single click — useMemo recomputes on every render.
```

**Explanation:** `fn` is a function defined inline in the component body. Every render creates a new function object. `Object.is(fn1, fn2)` is `false` even if both functions have identical code. So `fn` is always a "new" dep, and `useMemo` recomputes on every render — completely defeating memoization. Fix: wrap `fn` in `useCallback` with `[count]` as deps, so it only changes when `count` changes.

</details>

---

### Q10

```jsx
function App() {
  const [a, setA] = useState(1);
  const [b, setB] = useState(1);

  const product = useMemo(() => {
    console.log('product computed');
    return a * b;
  }, [a * b]); // computed value as dep

  return (
    <>
      <p>{product}</p>
      <button onClick={() => { setA(2); setB(1); }}>2 × 1</button>
      <button onClick={() => { setA(1); setB(2); }}>1 × 2</button>
    </>
  );
}
```

#### ❓ After the initial render, "2 × 1" is clicked then "1 × 2" is clicked. How many times does "product computed" log total?

<details>
<summary>✅ Answer</summary>

```txt
2 times: initial render, then after "2 × 1" click.
"1 × 2" does NOT trigger recomputation.
```

**Explanation:** The dep is `a * b` — a computed primitive value. After "2 × 1": `a=2, b=1`, so `a*b = 2`. `Object.is(1, 2) = false` → recomputes. After "1 × 2": `a=1, b=2`, so `a*b = 2`. `Object.is(2, 2) = true` → NO recomputation. The memoized value is based on the product, not the individual values. Logically the product is the same (2), so no recompute — even though `a` and `b` both changed.

</details>

---

## 3. useMemo vs No useMemo

### Q11

```jsx
const Child = React.memo(({ config }) => {
  console.log('Child rendered');
  return <p>{config.label}</p>;
});

function App() {
  const [count, setCount] = useState(0);

  const config = { label: 'hello' }; // no useMemo

  return (
    <>
      <Child config={config} />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </>
  );
}
```

#### ❓ After clicking the button 3 times, how many times does "Child rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
4 times (initial render + 3 button clicks)
```

**Explanation:** `React.memo` does a shallow comparison of props. `config` is a new object on every render (`{ label: 'hello' }` creates a new reference each time). Even though the content is identical, `Object.is(oldConfig, newConfig)` is `false`. So `React.memo` fails to bail out and re-renders `Child` on every parent render. The fix is `useMemo(() => ({ label: 'hello' }), [])` to stabilize the `config` reference.

</details>

---

### Q12

```jsx
const Child = React.memo(({ config }) => {
  console.log('Child rendered');
  return <p>{config.label}</p>;
});

function App() {
  const [count, setCount] = useState(0);

  const config = useMemo(() => ({ label: 'hello' }), []);

  return (
    <>
      <Child config={config} />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </>
  );
}
```

#### ❓ After clicking the button 3 times, how many times does "Child rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only the initial render)
```

**Explanation:** `useMemo(() => ({ label: 'hello' }), [])` with empty deps creates the config object once on mount and caches it forever. Every subsequent render returns the same object reference. `React.memo` compares `Object.is(oldConfig, newConfig)` — they are the same object, so `true` → `Child` does not re-render. Clicking the count button does not affect `Child` at all.

</details>

---

### Q13

```jsx
function App() {
  const [query, setQuery] = useState('');
  const [page, setPage] = useState(1);

  const searchParams = { query, page }; // no useMemo

  useEffect(() => {
    console.log('fetching with params:', searchParams);
    // fetchData(searchParams);
  }, [searchParams]);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <button onClick={() => setPage(p => p + 1)}>Next Page</button>
    </>
  );
}
```

#### ❓ What is the problem? When does the effect run?

<details>
<summary>✅ Answer</summary>

```txt
The effect runs on EVERY render, not just when query or page changes.
```

**Explanation:** `searchParams` is a new object on every render. `useEffect` with `[searchParams]` compares `Object.is(oldParams, newParams)` — they are always different objects (new reference each render), even if `query` and `page` haven't changed. So the effect runs after every single render, causing unnecessary fetch calls. Fix: `const searchParams = useMemo(() => ({ query, page }), [query, page])` to stabilize the reference, or put `query` and `page` directly in the deps array.

</details>

---

### Q14

```jsx
function App() {
  const [query, setQuery] = useState('');
  const [page, setPage] = useState(1);

  const searchParams = useMemo(
    () => ({ query, page }),
    [query, page]
  );

  useEffect(() => {
    console.log('fetching:', searchParams.query, searchParams.page);
  }, [searchParams]);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <button onClick={() => setPage(p => p + 1)}>Next Page</button>
    </>
  );
}
```

#### ❓ The input is changed to "react", then "Next Page" is clicked. How many times does the effect log?

<details>
<summary>✅ Answer</summary>

```txt
3 times:
1. On mount: "fetching:  1"
2. After typing "react": "fetching: react 1"
3. After "Next Page": "fetching: react 2"
```

**Explanation:** `searchParams` is now memoized. It only gets a new reference when `query` or `page` changes. The `useEffect` sees a changed dep (`searchParams`) only when the object is actually recreated. Three meaningful changes occur: mount, query change, and page change. No spurious fetches happen from unrelated re-renders.

</details>

---

### Q15

```jsx
function App() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');

  const processedName = useMemo(() => {
    return name.toUpperCase();
  }, [name]);

  useEffect(() => {
    console.log('name effect:', processedName);
  }, [processedName]);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>
      <button onClick={() => setName('Bob')}>Set Bob</button>
    </>
  );
}
```

#### ❓ After initial render, "Count" is clicked 3 times, then "Set Bob" once. How many times does the effect log?

<details>
<summary>✅ Answer</summary>

```txt
2 times:
1. On mount: "name effect: ALICE"
2. After "Set Bob": "name effect: BOB"
```

**Explanation:** `processedName` is memoized on `name`. Clicking "Count" changes `count` but not `name`, so `processedName` returns `'ALICE'` (cached). `useEffect` with `[processedName]` sees no change — it does not run. After "Set Bob", `name` changes, `processedName` recomputes to `'BOB'` (new primitive string), and the effect runs. `useMemo` here provides referential stability for the `useEffect` dep.

</details>

---

## 4. useMemo + React.memo

### Q16

```jsx
const ExpensiveList = React.memo(({ items }) => {
  console.log('ExpensiveList rendered, items count:', items.length);
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
});

function App() {
  const [theme, setTheme] = useState('light');
  const rawItems = [{ id: 1, name: 'Alpha' }, { id: 2, name: 'Beta' }];

  const items = useMemo(() => rawItems, []);

  return (
    <>
      <ExpensiveList items={items} />
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </>
  );
}
```

#### ❓ After toggling the theme 5 times, how many times does "ExpensiveList rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only the initial render)
```

**Explanation:** `items` is memoized with `[]` — it is the same array reference for the entire lifetime of `App`. Toggling theme re-renders `App` (changing `theme` state) but `items` never changes its reference. `React.memo` on `ExpensiveList` compares `Object.is(oldItems, newItems)` — they are the same array — so `ExpensiveList` never re-renders. Both `useMemo` and `React.memo` work together here.

</details>

---

### Q17

```jsx
const PriceTag = React.memo(({ price }) => {
  console.log('PriceTag rendered:', price);
  return <span>${price.toFixed(2)}</span>;
});

function App({ basePrice, taxRate }) {
  const [discount, setDiscount] = useState(0);

  const finalPrice = useMemo(
    () => basePrice * (1 + taxRate) * (1 - discount / 100),
    [basePrice, taxRate, discount]
  );

  return (
    <>
      <PriceTag price={finalPrice} />
      <button onClick={() => setDiscount(d => d + 5)}>Apply 5% discount</button>
    </>
  );
}
```

#### ❓ If `basePrice` and `taxRate` are stable (parent doesn't change them), and the discount button is clicked twice, how many times does "PriceTag rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
3 times: initial render + 2 discount clicks
```

**Explanation:** `finalPrice` is a number (primitive). Each time `discount` changes, `finalPrice` is recomputed to a new number value. Since the number changes, `React.memo` on `PriceTag` sees `Object.is(oldPrice, newPrice) = false` and re-renders. Even though `price` is a primitive (no referential equality issue), it is a different value — so `React.memo` correctly re-renders.

</details>

---

### Q18

```jsx
const Stats = React.memo(({ data }) => {
  console.log('Stats rendered');
  return <p>Count: {data.count}, Sum: {data.sum}</p>;
});

function App() {
  const [numbers, setNumbers] = useState([1, 2, 3]);
  const [theme, setTheme] = useState('light');

  const stats = useMemo(() => ({
    count: numbers.length,
    sum: numbers.reduce((a, b) => a + b, 0),
  }), [numbers]);

  return (
    <>
      <Stats data={stats} />
      <button onClick={() => setNumbers(n => [...n, n.length + 1])}>Add</button>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>Theme</button>
    </>
  );
}
```

#### ❓ "Theme" is clicked 3 times, then "Add" is clicked once. How many times does "Stats rendered" log total?

<details>
<summary>✅ Answer</summary>

```txt
2 times: initial render + 1 after "Add" click
```

**Explanation:** `stats` is memoized on `numbers`. Clicking "Theme" re-renders `App` but `numbers` is unchanged, so `stats` retains its reference. `React.memo` sees `Object.is(oldStats, newStats) = true` → `Stats` does not re-render. Clicking "Add" changes `numbers`, `stats` recomputes to a new object, and `Stats` re-renders. The combination of `useMemo` (value stability) + `React.memo` (render bail-out) works perfectly here.

</details>

---

### Q19

```jsx
const Button = React.memo(({ onClick, label }) => {
  console.log(`Button "${label}" rendered`);
  return <button onClick={onClick}>{label}</button>;
});

function App() {
  const [count, setCount] = useState(0);

  const handleClick = useMemo(() => () => {
    setCount(c => c + 1);
  }, []); // using useMemo to memoize a function

  return (
    <>
      <p>{count}</p>
      <Button onClick={handleClick} label="Increment" />
    </>
  );
}
```

#### ❓ Is using `useMemo` to memoize a function correct? How many times does "Button rendered" log after 3 clicks?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only initial render) — useMemo works here but useCallback is the idiomatic choice.
```

**Explanation:** `useMemo(() => () => setCount(c => c + 1), [])` returns the inner function. This is functionally equivalent to `useCallback(() => setCount(c => c + 1), [])`. The function is stable (same reference on every render). `React.memo` on `Button` sees `onClick` never changes and `label` is a stable string — so it never re-renders after the initial mount. Clicking the button changes `count` in `App` but `Button` is memoized and doesn't re-render. Note: `useCallback` is the idiomatic and clearer way to memoize functions.

</details>

---

### Q20

```jsx
const Child = React.memo(({ value }) => {
  console.log('Child rendered with:', value);
  return <p>{value.x}</p>;
});

function App() {
  const [n, setN] = useState(1);

  const value = useMemo(() => ({ x: n > 0 ? 'positive' : 'non-positive' }), [n > 0]);

  return (
    <>
      <Child value={value} />
      <button onClick={() => setN(n => n + 1)}>+</button>
    </>
  );
}
```

#### ❓ Starting from `n=1`, the button is clicked 3 times (n goes 1→2→3→4). How many times does "Child rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only the initial render)
```

**Explanation:** The dep is `n > 0` — a boolean expression. When `n=1`, `n > 0 = true`. When `n=2,3,4`, `n > 0` is still `true`. `Object.is(true, true) = true` — no recomputation. `value` is the same object reference. `React.memo` bails out — `Child` never re-renders. Note: using computed expressions as deps (like `n > 0`) is valid but can be confusing. It means "only recompute when the boolean `n > 0` changes." If `n` went from positive to 0 or negative, it would recompute.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function App() {
  const [count, setCount] = useState(0);

  const result = useMemo(() => {
    if (count === 0) return undefined;
    return count * 10;
  }, [count]);

  console.log('result:', result);

  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

#### ❓ What is logged for the initial render and after one click?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
result: undefined

// After first click (count = 1):
result: 10
```

**Explanation:** `useMemo` can return `undefined`. Returning `undefined` is perfectly valid — it is treated like any other cached value. On the initial render, `count = 0` so `result` is `undefined`. After clicking, `count = 1`, `useMemo` recomputes and returns `10`. There is no issue with `useMemo` returning `undefined`. The cached value is whatever the create function returns.

</details>

---

### Q22

```jsx
function App() {
  const [x, setX] = useState(1);

  const double = useMemo(() => {
    return () => x * 2;  // returns a function
  }, [x]);

  console.log('double()', double());

  return <button onClick={() => setX(v => v + 1)}>+</button>;
}
```

#### ❓ Is it valid for `useMemo` to return a function? How does this differ from `useCallback`?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render: double() → 2
// After click 1: double() → 4
// After click 2: double() → 6

Yes, it is valid. But useCallback is the idiomatic approach.
```

**Explanation:** `useMemo` caches whatever the create function returns — including a function. `useMemo(() => () => x * 2, [x])` is equivalent to `useCallback(() => x * 2, [x])`. Both create a new function when `x` changes. `useCallback` is syntactic sugar for this specific pattern. Using `useMemo` to return a function is correct but unusual and may confuse readers — `useCallback` is clearer in intent.

</details>

---

### Q23

```jsx
function App() {
  const [count, setCount] = useState(0);

  const a = useMemo(() => count + 1, [count]);
  const b = useMemo(() => a * 2, [a]);
  const c = useMemo(() => b + count, [b, count]);

  console.log('a:', a, 'b:', b, 'c:', c);

  return <button onClick={() => setCount(v => v + 1)}>+</button>;
}
```

#### ❓ Starting from count = 0, what is logged on initial render and after one click?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render (count = 0):
a: 1, b: 2, c: 2

// After click (count = 1):
a: 2, b: 4, c: 5
```

**Explanation:** This demonstrates chained `useMemo`:
- Initial: `a = 0+1 = 1`, `b = 1*2 = 2`, `c = 2+0 = 2`
- After click: `count = 1`, `a = 1+1 = 2`, `b = 2*2 = 4`, `c = 4+1 = 5`

Each `useMemo` in the chain recomputes only when its deps change. Since `a` depends on `count`, and `b` depends on `a`, and `c` depends on `b` and `count` — all three recompute when `count` changes. Chaining `useMemo` is valid for cascading derived computations.

</details>

---

### Q24

```jsx
let renderCount = 0;

function App() {
  const [x, setX] = useState(0);
  renderCount++;

  const expensive = useMemo(() => {
    return x * 1000;
  }, [x]);

  return (
    <>
      <p>expensive: {expensive}</p>
      <p>renders: {renderCount}</p>
      <button onClick={() => setX(v => v + 1)}>+</button>
    </>
  );
}
```

#### ❓ Does `useMemo` prevent the component from re-rendering? After 3 clicks, what is `renderCount`?

<details>
<summary>✅ Answer</summary>

```txt
renderCount: 4 (initial + 3 clicks)
useMemo does NOT prevent re-renders — it only caches the computed value within a render.
```

**Explanation:** `useMemo` does not prevent the component function from running. Every state change triggers a re-render (component function runs). `useMemo` only prevents the expensive computation from running on renders where deps are unchanged. To prevent re-rendering of the component itself, wrap it in `React.memo`. Here, every click changes `x` so `useMemo` recomputes anyway. `renderCount` increments on every render: 4 total.

</details>

---

### Q25

```jsx
function App() {
  const [count, setCount] = useState(0);

  // useMemo without dependency array — what happens?
  const result = useMemo(() => {
    console.log('computed');
    return count * 2;
  });

  return <button onClick={() => setCount(c => c + 1)}>{result}</button>;
}
```

#### ❓ Is omitting the dependency array from `useMemo` valid? What is the behavior?

<details>
<summary>✅ Answer</summary>

```txt
TypeScript/ESLint will warn. Behaviorally, useMemo without a deps array
recomputes on EVERY render — equivalent to no memoization at all.
```

**Explanation:** Unlike `useEffect`, `useMemo` without a deps array does not run once — it runs on every render. Without the array, React has no way to determine when to reuse the cached value, so it recomputes on every render. The `react-hooks/exhaustive-deps` ESLint rule requires the deps array. This is a mistake that provides zero performance benefit — just write `const result = count * 2;` directly. Always provide a deps array to `useMemo`.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Basic Memoization | Q1 – Q5 | when recomputes, batched updates, same-value bailout, object identity on recompute, Strict Mode double-invoke |
| Dependency Array | Q6 – Q10 | object dep instability, missing dep stale data, same-value dep bailout, function dep instability, computed dep expressions |
| useMemo vs No useMemo | Q11 – Q15 | React.memo + unstable ref, React.memo + stable ref, useEffect object dep problem, useEffect with memoized dep, memoized primitive dep |
| useMemo + React.memo | Q16 – Q20 | empty deps stability, primitive prop change, memoized object prevents re-render, useMemo to memoize function, computed boolean dep |
| Edge Cases | Q21 – Q25 | returning undefined, returning a function, chained useMemo, useMemo vs re-render prevention, missing deps array behavior |
