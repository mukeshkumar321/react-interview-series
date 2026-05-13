## useReducer — Tricky Output Questions

> These questions test your understanding of reducer state transitions, immutability, batching, dispatch from context, lazy initialization, and comparisons with `useState`. Each question reflects real scenarios from senior React interviews.

---

## 1. Basic Reducer

### Q1

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 };
    case 'DECREMENT': return { count: state.count - 1 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
    </>
  );
}
```

#### ❓ Starting from count = 0, "+", "+", "-" are clicked in sequence. What is the final count?

<details>
<summary>✅ Answer</summary>

```txt
1
```

**Explanation:** The reducer processes each dispatched action in order:
1. `INCREMENT`: `{ count: 0 + 1 }` → `{ count: 1 }`
2. `INCREMENT`: `{ count: 1 + 1 }` → `{ count: 2 }`
3. `DECREMENT`: `{ count: 2 - 1 }` → `{ count: 1 }`

Final state: `{ count: 1 }`. Each `dispatch` call triggers a re-render with the new state returned by the reducer.

</details>

---

### Q2

```jsx
function reducer(state, action) {
  console.log('reducer called, action:', action.type);
  switch (action.type) {
    case 'SET': return { value: action.payload };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { value: 'initial' });

  return (
    <>
      <p>{state.value}</p>
      <button onClick={() => dispatch({ type: 'SET', payload: 'updated' })}>
        Set
      </button>
    </>
  );
}
```

#### ❓ How many times is "reducer called" logged after clicking "Set" twice?

<details>
<summary>✅ Answer</summary>

```txt
2 times (one per click)
```

**Explanation:** Each `dispatch` call invokes the reducer once with the dispatched action. The reducer is not called on the initial render — only on dispatches. Note: In React 18 Strict Mode development, React double-invokes the reducer to verify it is pure. In production, one click = one reducer call. The reducer is a pure function: same state + same action = same result.

</details>

---

### Q3

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'A': return { ...state, a: state.a + 1 };
    case 'B': return { ...state, b: state.b + 1 };
    // missing default
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { a: 0, b: 0 });

  return (
    <>
      <p>a: {state.a}, b: {state.b}</p>
      <button onClick={() => dispatch({ type: 'C' })}>C</button>
    </>
  );
}
```

#### ❓ What happens when "C" is clicked?

<details>
<summary>✅ Answer</summary>

```txt
The reducer returns `undefined`. React sets state to `undefined`.
The component throws a TypeError: Cannot read properties of undefined (reading 'a').
```

**Explanation:** The `switch` statement has no `default` case. When `action.type = 'C'`, no case matches, and the function returns `undefined` implicitly. React sets the new state to `undefined`. On the next render, `state.a` throws a TypeError because `state` is `undefined`. The fix: always include a `default: return state;` case in reducers to handle unknown actions safely.

</details>

---

### Q4

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'ADD': return [...state, action.item];
    case 'CLEAR': return [];
    default: return state;
  }
}

function App() {
  const [items, dispatch] = useReducer(reducer, []);

  return (
    <>
      <p>Items: {items.length}</p>
      <button onClick={() => dispatch({ type: 'ADD', item: 'apple' })}>
        Add Apple
      </button>
      <button onClick={() => dispatch({ type: 'CLEAR' })}>Clear</button>
    </>
  );
}
```

#### ❓ "Add Apple" is clicked 3 times then "Clear". What does `items.length` show after each action?

<details>
<summary>✅ Answer</summary>

```txt
After 1st Add: 1
After 2nd Add: 2
After 3rd Add: 3
After Clear: 0
```

**Explanation:** The reducer handles array state immutably. `[...state, action.item]` creates a new array with the new item appended — never mutating the existing array. `CLEAR` returns a fresh empty array. Each `dispatch` call triggers a re-render with the new state. The component displays `items.length` correctly after each transition.

</details>

---

### Q5

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'TOGGLE':
      return { ...state, isActive: !state.isActive };
    case 'SET_NAME':
      return { ...state, name: action.payload };
    default:
      return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { name: 'Alice', isActive: false });

  return (
    <>
      <p>{state.name}: {state.isActive ? 'ON' : 'OFF'}</p>
      <button onClick={() => dispatch({ type: 'TOGGLE' })}>Toggle</button>
      <button onClick={() => dispatch({ type: 'SET_NAME', payload: 'Bob' })}>
        Set Bob
      </button>
    </>
  );
}
```

#### ❓ "Set Bob" is clicked, then "Toggle". What is displayed?

<details>
<summary>✅ Answer</summary>

```txt
Bob: ON
```

**Explanation:** Each action updates only the relevant field via spread:
1. `SET_NAME('Bob')`: `{ name: 'Bob', isActive: false }` — `name` updated, `isActive` preserved via `...state`
2. `TOGGLE`: `{ name: 'Bob', isActive: true }` — `isActive` flipped, `name` preserved via `...state`

The spread operator (`...state`) ensures all existing fields are preserved when only one field is updated. This is the correct immutable reducer pattern.

</details>

---

## 2. State Immutability

### Q6

```jsx
function reducer(state, action) {
  if (action.type === 'SAME') return state; // same reference
  return { ...state };
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  console.log('rendered');

  return (
    <>
      <button onClick={() => dispatch({ type: 'SAME' })}>Same</button>
      <button onClick={() => dispatch({ type: 'NEW' })}>New</button>
    </>
  );
}
```

#### ❓ "Same" is clicked twice, "New" is clicked once. How many times does "rendered" log after the initial render?

<details>
<summary>✅ Answer</summary>

```txt
Same click 1: 0 additional renders (or possibly 1 for bailout check, then no more)
Same click 2: 0 additional renders
New click: 1 render

Total after initial: ~1 render
```

**Explanation:** When the reducer returns the exact same state object reference (`return state`), React performs a bailout — `Object.is(oldState, newState) = true` → no re-render. The first `SAME` dispatch may trigger one component execution to confirm the bailout, but after that, subsequent `SAME` dispatches skip even that. `NEW` returns a new object (`{ ...state }`), even though values are identical. `Object.is` compares references → new object → re-render. This shows that returning the same reference from a reducer is a reliable optimization.

</details>

---

### Q7

```jsx
function reducer(state, action) {
  if (action.type === 'PUSH') {
    state.items.push(action.item); // direct mutation!
    return state; // same reference
  }
  return state;
}

function App() {
  const [state, dispatch] = useReducer(reducer, { items: [] });

  return (
    <>
      <p>Items: {state.items.length}</p>
      <button onClick={() => dispatch({ type: 'PUSH', item: 'a' })}>Push</button>
    </>
  );
}
```

#### ❓ After clicking "Push" 3 times, what does `state.items.length` display?

<details>
<summary>✅ Answer</summary>

```txt
0 (the UI does not update)
```

**Explanation:** The reducer mutates `state.items` directly with `.push()`, then returns the same `state` reference. React's `Object.is(oldState, newState) = true` (same reference) → bailout. No re-render occurs. The internal array has 3 items (`state.items` was mutated), but React never sees a state change, so the component never re-renders to show the updated count. Mutating state in a reducer and returning the same reference is a critical bug — always return a new object/array.

</details>

---

### Q8

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'UPDATE_USER':
      state.user.name = action.name; // mutation!
      return { ...state }; // new top-level object but mutated user
    default:
      return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, {
    user: { name: 'Alice' },
    theme: 'light',
  });

  return (
    <>
      <p>{state.user.name}</p>
      <button onClick={() => dispatch({ type: 'UPDATE_USER', name: 'Bob' })}>
        Update
      </button>
    </>
  );
}
```

#### ❓ Does the component re-render? What are the risks of this approach?

<details>
<summary>✅ Answer</summary>

```txt
Yes, the component re-renders (new top-level object reference).
But the user object is mutated — this can cause bugs with memoized children.
```

**Explanation:** `{ ...state }` creates a new top-level object, so `Object.is(oldState, newState) = false` → re-render occurs. The displayed name updates to `'Bob'`. However, `state.user` is the same mutated object reference. Any `React.memo`-wrapped child that receives `state.user` as a prop would see the same reference and bail out — displaying stale data (it already has the mutation, but React won't know to re-render it). Always create a new nested object: `{ ...state, user: { ...state.user, name: action.name } }`.

</details>

---

### Q9

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'ADD':
      return { ...state, list: [...state.list, action.item] };
    case 'REMOVE':
      return { ...state, list: state.list.filter(i => i !== action.item) };
    case 'CLEAR':
      return { ...state, list: [] };
    default:
      return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { list: ['a', 'b', 'c'], count: 0 });

  return (
    <>
      <p>{state.list.join(', ')}</p>
      <button onClick={() => dispatch({ type: 'REMOVE', item: 'b' })}>Remove b</button>
    </>
  );
}
```

#### ❓ After clicking "Remove b", what does the `<p>` display? Is `state.count` preserved?

<details>
<summary>✅ Answer</summary>

```txt
a, c
Yes — state.count is preserved (still 0)
```

**Explanation:** `filter` returns a new array without `'b'`. `{ ...state, list: ... }` spreads all existing fields (`count`, etc.) and overrides only `list`. So `count: 0` is preserved. The `<p>` displays `'a, c'`. This demonstrates the correct immutable update pattern for arrays in reducers — `filter` (remove), `map` (update), `[...arr, item]` (add) all return new arrays without mutating the original.

</details>

---

### Q10

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'SET_ACTIVE':
      return state.map(user =>
        user.id === action.id
          ? { ...user, active: true }
          : { ...user, active: false }
      );
    default:
      return state;
  }
}

function App() {
  const [users, dispatch] = useReducer(reducer, [
    { id: 1, name: 'Alice', active: false },
    { id: 2, name: 'Bob', active: false },
  ]);

  return (
    <>
      {users.map(u => (
        <p key={u.id}>{u.name}: {u.active ? 'active' : 'inactive'}</p>
      ))}
      <button onClick={() => dispatch({ type: 'SET_ACTIVE', id: 1 })}>
        Activate Alice
      </button>
    </>
  );
}
```

#### ❓ After clicking "Activate Alice", what does the list show?

<details>
<summary>✅ Answer</summary>

```txt
Alice: active
Bob: inactive
```

**Explanation:** `state.map(...)` returns a new array. For `id === 1` (Alice), a new object with `active: true` is created. For all others, a new object with `active: false` is created. This ensures only one user is active at a time and all user objects are new references. The result is immutable and correctly reflects the intended state.

</details>

---

## 3. Multiple Dispatches

### Q11

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    case 'DOUBLE': return { count: state.count * 2 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 1 });

  const handleClick = () => {
    dispatch({ type: 'INC' });
    dispatch({ type: 'DOUBLE' });
  };

  console.log('render, count:', state.count);

  return <button onClick={handleClick}>{state.count}</button>;
}
```

#### ❓ What is the count after one button click? How many times does "render" log?

<details>
<summary>✅ Answer</summary>

```txt
count: 4
renders: 2 total (initial + 1 after click)
```

**Explanation:** React batches both dispatches into a single re-render in React 18. The reducer processes them sequentially:
1. `INC`: `{ count: 1 + 1 }` → `{ count: 2 }`
2. `DOUBLE`: `{ count: 2 * 2 }` → `{ count: 4 }`

The component re-renders once with the final state `{ count: 4 }`. "render" logs twice total: initial (count: 1) and after click (count: 4).

</details>

---

### Q12

```jsx
function reducer(state, action) {
  console.log('reducer, action:', action.type, 'state before:', state.count);
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  const handleClick = () => {
    dispatch({ type: 'INC' });
    dispatch({ type: 'INC' });
  };

  return <button onClick={handleClick}>{state.count}</button>;
}
```

#### ❓ What is logged when the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
reducer, action: INC state before: 0
reducer, action: INC state before: 1
```

**Explanation:** Both dispatches are processed synchronously before the re-render. The reducer is called twice. The first call receives `state.count = 0` and returns `{ count: 1 }`. The second call receives the result of the first — `state.count = 1` — and returns `{ count: 2 }`. After both reducer calls, React re-renders once with `{ count: 2 }`. This confirms that multiple dispatches in a batch each receive the updated state from the previous reducer call.

</details>

---

### Q13

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'STEP1': return { phase: 'loading' };
    case 'STEP2': return { phase: 'done' };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { phase: 'idle' });

  console.log('phase:', state.phase);

  const handleClick = () => {
    dispatch({ type: 'STEP1' });
    dispatch({ type: 'STEP2' });
  };

  return <button onClick={handleClick}>Start</button>;
}
```

#### ❓ After clicking "Start", what phases are logged and in what order?

<details>
<summary>✅ Answer</summary>

```txt
phase: idle        ← initial render
phase: done        ← single re-render after click
```

**Explanation:** React 18 batches both dispatches. The reducer processes them in sequence: `STEP1` → `{ phase: 'loading' }`, then `STEP2` → `{ phase: 'done' }`. React applies the final state in a single re-render. The `'loading'` phase is never visible to the user or logged — the component jumps directly from `'idle'` to `'done'`. If you need to see intermediate states, you need two separate event loop ticks (e.g., using `setTimeout`).

</details>

---

### Q14

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  useEffect(() => {
    dispatch({ type: 'INC' });
    dispatch({ type: 'INC' });
  }, []);

  console.log('count:', state.count);

  return <p>{state.count}</p>;
}
```

#### ❓ What is logged in order from initial render through effects?

<details>
<summary>✅ Answer</summary>

```txt
count: 0     ← initial render
count: 2     ← single re-render after effect dispatches
```

**Explanation:** The initial render logs `count: 0`. After mount, the effect fires and dispatches `INC` twice. React batches both dispatches (React 18 batches even inside `useEffect`). The reducer processes both: `0 → 1 → 2`. A single re-render occurs with `count: 2`. Note: in Strict Mode, the effect runs twice (mount, cleanup, remount), so count would end at `4`.

</details>

---

### Q15

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'RESET': return { count: 0 };
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 5 });

  const handleClick = () => {
    dispatch({ type: 'RESET' });
    dispatch({ type: 'INC' });
    dispatch({ type: 'INC' });
  };

  return <button onClick={handleClick}>{state.count}</button>;
}
```

#### ❓ What is the count after clicking the button?

<details>
<summary>✅ Answer</summary>

```txt
2
```

**Explanation:** The three dispatches are processed sequentially:
1. `RESET`: `{ count: 0 }` — count is reset
2. `INC`: `{ count: 0 + 1 }` = `{ count: 1 }`
3. `INC`: `{ count: 1 + 1 }` = `{ count: 2 }`

React batches all three into one re-render with the final state `{ count: 2 }`. The initial `count: 5` is lost because RESET was applied first, then INC twice from 0.

</details>

---

## 4. useReducer + useContext

### Q16

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
};

const StoreContext = createContext(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <StoreContext.Provider value={{ state, dispatch }}>
      {children}
    </StoreContext.Provider>
  );
}

function Display() {
  const { state } = useContext(StoreContext);
  console.log('Display:', state.count);
  return <p>{state.count}</p>;
}

function Button() {
  const { dispatch } = useContext(StoreContext);
  console.log('Button rendered');
  return (
    <button onClick={() => dispatch({ type: 'INC' })}>+</button>
  );
}

function App() {
  return (
    <Provider>
      <Display />
      <Button />
    </Provider>
  );
}
```

#### ❓ After clicking "+" once, how many times does each component log?

<details>
<summary>✅ Answer</summary>

```txt
Display: 2 times (initial: 0, after click: 1)
Button: 2 times (initial render + re-render due to context object change)
```

**Explanation:** Dispatching `INC` triggers Provider to re-render with new state. The context value `{ state, dispatch }` is a new object on each render (even though `dispatch` is stable). `Object.is(oldValue, newValue) = false` → all consumers re-render. `Display` re-renders to show the new count. `Button` also re-renders unnecessarily (its output hasn't changed). Fix: split contexts (ValueContext + DispatchContext) and wrap `Button` in `React.memo`.

</details>

---

### Q17

```jsx
const reducer = (state, action) => {
  if (action.type === 'TOGGLE') return { on: !state.on };
  return state;
};

const DispatchContext = createContext(null);
const StateContext = createContext({ on: false });

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, { on: false });
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  );
}

const Toggle = React.memo(() => {
  const dispatch = useContext(DispatchContext);
  console.log('Toggle rendered');
  return (
    <button onClick={() => dispatch({ type: 'TOGGLE' })}>Toggle</button>
  );
});

function Indicator() {
  const { on } = useContext(StateContext);
  console.log('Indicator:', on);
  return <p>{on ? 'ON' : 'OFF'}</p>;
}
```

#### ❓ After clicking "Toggle" 3 times, how many times does "Toggle rendered" log?

<details>
<summary>✅ Answer</summary>

```txt
1 time (only the initial render)
```

**Explanation:** `dispatch` from `useReducer` is stable across renders (same reference guaranteed by React). `DispatchContext.Provider value={dispatch}` provides the same reference every time. `Toggle` is wrapped in `React.memo` with no props — neither its props nor the `DispatchContext` value changes. React.memo prevents parent-triggered re-renders, and the context doesn't change. This is the optimal split-context + `React.memo` pattern.

</details>

---

### Q18

```jsx
const CountContext = createContext(0);

const reducer = (count, action) => {
  if (action.type === 'INC') return count + 1;
  return count;
};

function App() {
  const [count, dispatch] = useReducer(reducer, 0);

  return (
    <CountContext.Provider value={count}>
      <Child dispatch={dispatch} />
    </CountContext.Provider>
  );
}

function Child({ dispatch }) {
  const count = useContext(CountContext);
  console.log('Child count:', count);
  return (
    <button onClick={() => dispatch({ type: 'INC' })}>{count}</button>
  );
}
```

#### ❓ After clicking the button 2 times, how many times does "Child count" log total?

<details>
<summary>✅ Answer</summary>

```txt
3 times (initial: 0, click 1: 1, click 2: 2)
```

**Explanation:** `Child` re-renders when the context value changes. Clicking the button dispatches `INC`, which changes `count` in `App`. `App` re-renders with new `count`, which is passed as the context value. `Child` subscribes to the context via `useContext(CountContext)` → re-renders on every count change. Each re-render logs the current count. `dispatch` is passed as a prop; since it's stable, `React.memo` could bail out on `dispatch` changes — but there's no `React.memo` here.

</details>

---

### Q19

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'ADD_TODO':
      return { todos: [...state.todos, action.todo] };
    case 'TOGGLE_TODO':
      return {
        todos: state.todos.map(t =>
          t.id === action.id ? { ...t, done: !t.done } : t
        )
      };
    default:
      return state;
  }
};

const TodoContext = createContext(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, { todos: [] });
  return (
    <TodoContext.Provider value={{ ...state, dispatch }}>
      {children}
    </TodoContext.Provider>
  );
}

function TodoList() {
  const { todos } = useContext(TodoContext);
  console.log('TodoList rendered, count:', todos.length);
  return <ul>{todos.map(t => <li key={t.id}>{t.text}</li>)}</ul>;
}
```

#### ❓ After adding 2 todos, how many times does "TodoList rendered" log total?

<details>
<summary>✅ Answer</summary>

```txt
3 times (initial: 0, after 1st add: 1, after 2nd add: 2)
```

**Explanation:** Each `ADD_TODO` dispatch causes `Provider` to re-render with new state. The context value `{ ...state, dispatch }` is a new object each time (spread creates new). `Object.is` sees a new reference → `TodoList` re-renders. The initial render with empty todos, and one re-render per add operation = 3 total renders for 2 adds.

</details>

---

### Q20

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'SET': return action.value;
    default: return state;
  }
};

const Ctx = createContext(null);

function Parent() {
  const [value, dispatch] = useReducer(reducer, 'hello');
  return (
    <Ctx.Provider value={{ value, dispatch }}>
      <Consumer />
    </Ctx.Provider>
  );
}

function Consumer() {
  const { value, dispatch } = useContext(Ctx);
  console.log('Consumer:', value);

  useEffect(() => {
    dispatch({ type: 'SET', value: 'world' });
  }, []); // runs once on mount

  return <p>{value}</p>;
}
```

#### ❓ What sequence of logs appears?

<details>
<summary>✅ Answer</summary>

```txt
Consumer: hello    ← initial render
Consumer: world    ← re-render after effect dispatch
```

**Explanation:** `Consumer` renders initially with `value = 'hello'`. After mount, the effect fires and dispatches `SET` with `'world'`. `Parent` re-renders with new state `'world'`. Context value changes → `Consumer` re-renders and logs `'world'`. The effect runs only once (empty deps) so no infinite loop. Note: `dispatch` is stable (won't change between renders), so even if it were in the deps array, the effect would still run only once.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, undefined, () => {
    console.log('lazy init');
    return { count: 10 };
  });

  console.log('render');

  return <button onClick={() => dispatch({ type: 'INC' })}>{state.count}</button>;
}
```

#### ❓ What is logged on the initial render? After one click?

<details>
<summary>✅ Answer</summary>

```txt
// Initial render:
lazy init
render

// After one click:
render
```

**Explanation:** The third argument to `useReducer` is a lazy initializer function. It receives the second argument (`undefined`) as its parameter and its return value becomes the initial state. It is called only once — on the first render. After clicks, the initializer is never called again. `render` logs on every render. The button starts showing `10` and increments to `11` after one click.

</details>

---

### Q22

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { count: 0 }, (init) => {
    console.log('init called with:', init);
    return { count: init.count + 100 };
  });

  return <p>{state.count}</p>;
}
```

#### ❓ What is logged and what is the initial count displayed?

<details>
<summary>✅ Answer</summary>

```txt
init called with: { count: 0 }
Displayed: 100
```

**Explanation:** The third argument to `useReducer` is an initializer function that receives the second argument (`{ count: 0 }`) as its parameter. The function returns `{ count: 0 + 100 }` = `{ count: 100 }`. This is the initial state. This pattern is useful for expensive initializations — the function runs once. In Strict Mode development, React double-invokes it to check purity: "init called with: { count: 0 }" would appear twice.

</details>

---

### Q23

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'UPDATE': return { value: action.value };
    default: return state;
  }
}

function App() {
  const [state, dispatch] = useReducer(reducer, { value: 'initial' });

  useEffect(() => {
    dispatch({ type: 'UPDATE', value: 'from effect' });

    return () => {
      dispatch({ type: 'UPDATE', value: 'from cleanup' });
    };
  }, []);

  console.log('render:', state.value);

  return <p>{state.value}</p>;
}
```

#### ❓ In Strict Mode development, what sequence of logs appears?

<details>
<summary>✅ Answer</summary>

```txt
render: initial           ← first mount render
render: initial           ← StrictMode double mount render
render: from cleanup      ← re-render from cleanup dispatch (StrictMode unmount)
render: from effect       ← re-render from effect dispatch (StrictMode remount)
```

**Explanation:** React 18 Strict Mode mounts, unmounts, and remounts components. The sequence: (1) mount → effect fires → `'from effect'` dispatch. (2) StrictMode forced unmount → cleanup fires → `'from cleanup'` dispatch. (3) StrictMode remount → effect fires again → `'from effect'` dispatch. Each dispatch causes a re-render. This demonstrates why cleanup functions should reverse side effects — dispatching in cleanup can cause unexpected state transitions. In production, only `'initial'` and `'from effect'` render.

</details>

---

### Q24

```jsx
// useState version
function CounterState() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

// useReducer version
const incReducer = (state) => state + 1;

function CounterReducer() {
  const [count, dispatch] = useReducer(incReducer, 0);
  return <button onClick={dispatch}>{count}</button>;
}
```

#### ❓ Are these two implementations equivalent? What is passed to the `dispatch` function when the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
They are functionally equivalent for this use case.
When the button is clicked, the SyntheticEvent object is passed to dispatch as the action.
```

**Explanation:** Both counters increment by 1 on each click and re-render. In `CounterReducer`, `onClick={dispatch}` passes the click event (a SyntheticEvent) directly as the action to the reducer. The reducer ignores the action and simply returns `state + 1`. This works because `incReducer` doesn't use the action. In practice, using `dispatch` directly as an event handler is clever but uncommon — it only works when the action/event payload is irrelevant.

</details>

---

### Q25

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'INC': return { count: state.count + 1 };
    default: return state;
  }
}

function App() {
  const [count, setCount] = useState(0);
  const [state, dispatch] = useReducer(reducer, { count: 0 });

  const isStateAhead = state.count > count;

  return (
    <>
      <p>useState: {count}</p>
      <p>useReducer: {state.count}</p>
      <p>Reducer ahead: {isStateAhead ? 'yes' : 'no'}</p>
      <button onClick={() => setCount(c => c + 1)}>State +</button>
      <button onClick={() => dispatch({ type: 'INC' })}>Reducer +</button>
    </>
  );
}
```

#### ❓ "Reducer +" is clicked 3 times, then "State +" is clicked 1 time. What is displayed?

<details>
<summary>✅ Answer</summary>

```txt
useState: 1
useReducer: 3
Reducer ahead: yes
```

**Explanation:** `useState` and `useReducer` are independent pieces of state. Clicking "Reducer +" 3 times increments `state.count` to `3` (via reducer). Clicking "State +" increments `count` to `1` (via setState). They don't affect each other. `isStateAhead` checks `3 > 1 = true`. Both hooks can coexist in the same component, each managing their own independent state.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Basic Reducer | Q1 – Q5 | sequential state transitions, reducer call count, missing default case crash, array state, spread operator preserves fields |
| State Immutability | Q6 – Q10 | same reference bailout, mutation + same reference = no re-render, nested object mutation with shallow spread, array immutability, map for array update |
| Multiple Dispatches | Q11 – Q15 | batch processing, reducer gets previous result, intermediate states hidden in batch, dispatches in useEffect, RESET then INC |
| useReducer + useContext | Q16 – Q20 | combined object causes all consumers to re-render, stable dispatch reference, split context + React.memo, dispatch in useEffect from consumer, context value object spreading |
| Edge Cases | Q21 – Q25 | lazy initializer third argument, initializer receives second arg, dispatch in cleanup (StrictMode), dispatch as event handler, useState vs useReducer independence |
