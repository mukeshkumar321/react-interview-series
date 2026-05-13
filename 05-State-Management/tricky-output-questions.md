## State Management — Tricky Output Questions

> These questions test your understanding of Context API re-render behavior, Redux Toolkit's Immer-based mutation, Zustand selector subscriptions, state colocation decisions, and derived state patterns. Each question reflects a real performance issue or interview trap.

---

## 1. Context API Performance

### Q1

```jsx
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  const [fontSize, setFontSize] = useState(16);

  return (
    <ThemeContext.Provider value={{ theme, setTheme, fontSize, setFontSize }}>
      {children}
    </ThemeContext.Provider>
  );
}

function ThemeButton() {
  const { theme } = useContext(ThemeContext);
  console.log("ThemeButton renders");
  return <button>{theme}</button>;
}

function FontSizeDisplay() {
  const { fontSize } = useContext(ThemeContext);
  console.log("FontSizeDisplay renders");
  return <p>Font: {fontSize}px</p>;
}
```

#### ❓ `setFontSize(18)` is called. Which components re-render?

<details>
<summary>✅ Answer</summary>

```txt
ThemeButton renders
FontSizeDisplay renders
```

**Explanation:** Both components re-render even though `ThemeButton` only uses `theme` and `theme` did not change. This is the core Context API performance limitation. When `setFontSize` is called, `ThemeProvider` re-renders, creating a new object `{ theme, setTheme, fontSize, setFontSize }`. This new object reference causes every context consumer to re-render, regardless of which property they actually use. Context does not support partial subscriptions — every consumer re-renders when any part of the value changes.

</details>

---

### Q2

```jsx
const CountContext = createContext();

function App() {
  const [count, setCount] = useState(0);

  return (
    <CountContext.Provider value={{ count, setCount }}>
      <Display />
      <Incrementor />
    </CountContext.Provider>
  );
}

function Display() {
  const { count } = useContext(CountContext);
  console.log("Display renders");
  return <p>{count}</p>;
}

function Incrementor() {
  const { setCount } = useContext(CountContext);
  console.log("Incrementor renders");
  return <button onClick={() => setCount(c => c + 1)}>+</button>;
}
```

#### ❓ On every button click, which components log to the console?

<details>
<summary>✅ Answer</summary>

```txt
Display renders
Incrementor renders
```

**Explanation:** Both `Display` and `Incrementor` re-render on every count change. Even though `Incrementor` only uses `setCount` (which is stable and never changes), it still subscribes to the entire context value object. The context value `{ count, setCount }` is a new object reference on every render of `App`. To fix: split the context into a `CountValueContext` (count) and `CountDispatchContext` (setCount). `Incrementor` subscribes only to `CountDispatchContext`, which receives a stable value and does not change.

</details>

---

### Q3

```jsx
const UserContext = createContext(null);

function UserProvider({ children }) {
  const [user, setUser] = useState({ name: "Alice", role: "admin" });

  const updateName = useCallback((name) => {
    setUser(prev => ({ ...prev, name }));
  }, []);

  const contextValue = useMemo(() => ({ user, updateName }), [user, updateName]);

  return (
    <UserContext.Provider value={contextValue}>
      {children}
    </UserContext.Provider>
  );
}
```

#### ❓ Does using `useMemo` on `contextValue` prevent unnecessary re-renders of context consumers?

<details>
<summary>✅ Answer</summary>

```txt
Yes, partially — it prevents re-renders when UserProvider re-renders
for reasons unrelated to user or updateName changes.

But when user state changes (which is the common case), contextValue
is a new object and all consumers still re-render.
```

**Explanation:** `useMemo(() => ({ user, updateName }), [user, updateName])` only returns a new object when `user` or `updateName` changes. If `UserProvider` re-renders for another reason (e.g., a sibling state change), the memoized value stays the same, preventing unnecessary consumer re-renders. However, any time `user` changes — which is the typical case — all consumers still re-render. `useMemo` is still worthwhile for stability but is not a cure-all for context performance.

</details>

---

### Q4

```jsx
const Ctx = createContext(0);

function Parent() {
  const [count, setCount] = useState(0);
  return (
    <Ctx.Provider value={count}>
      <Child />
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </Ctx.Provider>
  );
}

const Child = React.memo(() => {
  const count = useContext(Ctx);
  console.log("Child renders:", count);
  return <p>{count}</p>;
});
```

#### ❓ Does `React.memo` prevent `Child` from re-rendering when the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
No — Child still re-renders on every click.
```

**Explanation:** `React.memo` prevents re-renders caused by parent re-renders when props do not change. But `Child` re-renders because it is a context consumer — when the context value changes, all consumers re-render regardless of `React.memo`. The subscription to `useContext` bypasses `memo`'s prop comparison. `memo` + `useContext` does not give you selective re-rendering based on which part of the context changed.

</details>

---

### Q5

```jsx
const AppContext = createContext();

function AppProvider({ children }) {
  const [user, setUser] = useState(null);
  const [cart, setCart] = useState([]);
  const [notifications, setNotifications] = useState([]);

  return (
    <AppContext.Provider value={{ user, setUser, cart, setCart, notifications, setNotifications }}>
      {children}
    </AppContext.Provider>
  );
}

function CartIcon() {
  const { cart } = useContext(AppContext);
  console.log("CartIcon renders");
  return <span>{cart.length}</span>;
}
```

#### ❓ A new notification arrives and `setNotifications([...notifications, newNotif])` is called. Does `CartIcon` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — CartIcon re-renders even though cart did not change.
```

**Explanation:** The `AppContext.Provider` receives a new value object (new reference) every time `AppProvider` re-renders. `setNotifications` triggers a re-render of `AppProvider`, which creates a new `{ user, setUser, cart, setCart, notifications, setNotifications }` object. Every consumer of `AppContext` — including `CartIcon` — re-renders. The solution is to split the monolithic context into separate contexts: `UserContext`, `CartContext`, `NotificationsContext`. Then changing notifications only re-renders notification consumers.

</details>

---

## 2. Redux and Redux Toolkit

### Q6

```jsx
// Redux Toolkit slice
const counterSlice = createSlice({
  name: "counter",
  initialState: { value: 0 },
  reducers: {
    increment: (state) => {
      state.value += 1;       // looks like mutation
    },
    addAmount: (state, action) => {
      state.value += action.payload;
    },
  },
});
```

#### ❓ The `increment` reducer mutates `state.value` directly. Is this valid? Would this be valid in plain Redux?

<details>
<summary>✅ Answer</summary>

```txt
Valid in Redux Toolkit — Immer makes it safe.
NOT valid in plain Redux — mutations break state comparison and cause bugs.
```

**Explanation:** Redux Toolkit uses Immer under the hood. Immer wraps the state in a Proxy. When you write `state.value += 1`, Immer intercepts the mutation, records it, and produces a new immutable state object. Your reducer looks like a mutation but Immer converts it to an immutable update. In plain Redux (without Immer), you must return a new object: `return { ...state, value: state.value + 1 }`. Mutating state directly in plain Redux causes `===` comparisons to see no change, breaking re-renders and memoized selectors.

</details>

---

### Q7

```jsx
const counterSlice = createSlice({
  name: "counter",
  initialState: { value: 0 },
  reducers: {
    increment: (state) => {
      state.value += 1;
    },
    reset: (state) => {
      state = { value: 0 };   // reassigning the parameter
    },
  },
});
```

#### ❓ Does the `reset` reducer correctly reset the state to `{ value: 0 }`?

<details>
<summary>✅ Answer</summary>

```txt
No — reset does NOT work. State remains unchanged.
```

**Explanation:** Immer tracks mutations through the Proxy reference. Reassigning the local `state` parameter (`state = { value: 0 }`) replaces the local variable reference but does not mutate the Proxy — Immer sees no changes. To reset state, you have two correct options: (1) mutate via the proxy: `state.value = 0`; (2) return a new state object explicitly: `return { value: 0 }`. When you return a value from an Immer reducer, Immer uses that returned value as the new state instead of applying recorded mutations.

</details>

---

### Q8

```jsx
const selectTotal = (state) => state.cart.items.reduce(
  (sum, item) => sum + item.price * item.quantity, 0
);

function CartTotal() {
  const total = useSelector(selectTotal);
  console.log("CartTotal renders");
  return <p>Total: ${total}</p>;
}
```

#### ❓ An unrelated piece of state (e.g., `user.name`) changes in the Redux store. Does `CartTotal` re-render?

<details>
<summary>✅ Answer</summary>

```txt
No — CartTotal does NOT re-render if the total value is the same.
```

**Explanation:** `useSelector` runs the selector function on every state change. After `user.name` changes, `selectTotal` runs again. If `state.cart.items` is unchanged, the `reduce` returns the same number. `useSelector` uses strict equality (`===`) to compare the previous and new return values. Since the numbers are equal (`42 === 42`), no re-render is triggered. If the selector returned an object or array, a new reference would trigger a re-render even with the same contents — that's when `createSelector` (reselect) becomes necessary.

</details>

---

### Q9

```jsx
const selectUserProfile = (state) => ({
  name: state.user.name,
  email: state.user.email,
});

function Profile() {
  const profile = useSelector(selectUserProfile);
  console.log("Profile renders");
  return <p>{profile.name}</p>;
}
```

#### ❓ An unrelated part of the store changes. Does `Profile` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — Profile re-renders on every store change.
```

**Explanation:** `selectUserProfile` returns a new object `{ name: ..., email: ... }` on every call, even if `name` and `email` are unchanged. `useSelector` compares the previous return value with the new one using `===`. A new object is never `===` to a previous object, so `useSelector` always sees a "change" and triggers a re-render. Fix: use `createSelector` from `reselect` (which memoizes the output) or use `shallowEqual` as the second argument to `useSelector`: `useSelector(selectUserProfile, shallowEqual)`.

</details>

---

### Q10

```jsx
const store = configureStore({ reducer: rootReducer });

store.dispatch({ type: "counter/increment" });
store.dispatch({ type: "counter/increment" });
store.dispatch({ type: "counter/increment" });

console.log(store.getState().counter.value);
```

#### ❓ What does `console.log` output?

<details>
<summary>✅ Answer</summary>

```txt
3
```

**Explanation:** `store.dispatch` is synchronous in Redux. Each dispatch immediately runs the reducer and updates the state. After three dispatches, the counter value has been incremented three times. `store.getState()` always returns the current state synchronously. There is no batching of dispatches in Redux itself — each dispatch is a separate, immediate state update. (React does batch re-renders in components, but the store state update happens synchronously for each dispatch.)

</details>

---

## 3. Zustand

### Q11

```jsx
const useStore = create((set) => ({
  bears: 0,
  fish: 0,
  addBear: () => set((state) => ({ bears: state.bears + 1 })),
  addFish: () => set((state) => ({ fish: state.fish + 1 })),
}));

function Bears() {
  const bears = useStore((state) => state.bears);
  console.log("Bears renders");
  return <p>Bears: {bears}</p>;
}

function Fish() {
  const fish = useStore((state) => state.fish);
  console.log("Fish renders");
  return <p>Fish: {fish}</p>;
}
```

#### ❓ `addBear()` is called. Which components re-render?

<details>
<summary>✅ Answer</summary>

```txt
Bears renders
Fish does NOT re-render.
```

**Explanation:** Zustand uses selector subscriptions. `Bears` subscribes to `state => state.bears` and re-renders only when `bears` changes. `Fish` subscribes to `state => state.fish` and only re-renders when `fish` changes. This is fundamentally different from Context API — Zustand supports fine-grained subscriptions out of the box. When `addBear` updates `bears`, Zustand checks each subscriber's selector. Only `Bears`'s selector returns a different value, so only `Bears` re-renders.

</details>

---

### Q12

```jsx
const useStore = create((set) => ({
  user: { name: "Alice", role: "admin" },
  updateName: (name) => set((state) => ({
    user: { ...state.user, name },
  })),
}));

function UserCard() {
  const user = useStore((state) => state.user);
  console.log("UserCard renders");
  return <p>{user.name} - {user.role}</p>;
}
```

#### ❓ `updateName("Bob")` is called. Does `UserCard` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — UserCard re-renders.
```

**Explanation:** The selector `state => state.user` returns the `user` object. After `updateName("Bob")`, the state contains a new `user` object `{ name: "Bob", role: "admin" }`. Zustand compares the previous selector result with the new one using `Object.is` (strict equality). The previous `user` object and the new `user` object are different references, so `Object.is(prevUser, newUser)` is `false`. Re-render triggered. To be more selective, use a more specific selector: `state => state.user.name`.

</details>

---

### Q13

```jsx
const useStore = create((set) => ({
  items: ["a", "b", "c"],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
}));

function ItemList() {
  const items = useStore(
    (state) => state.items,
    (a, b) => a.length === b.length  // custom equality
  );
  console.log("ItemList renders");
  return <ul>{items.map(i => <li key={i}>{i}</li>)}</ul>;
}
```

#### ❓ `addItem("d")` is called (adding a 4th item). Does `ItemList` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — ItemList re-renders.
The length goes from 3 to 4, so the custom equality returns false.
```

**Explanation:** Zustand allows a custom equality function as the second argument to the selector. The custom equality `(a, b) => a.length === b.length` only prevents re-renders when the array length does not change. After adding `"d"`, the new array has length 4. The old array had length 3. `3 === 4` is `false`, so the component re-renders. If someone replaced an item without changing the length (e.g., `items[0] = "x"`), this custom equality would prevent re-render even though the contents changed — potentially a bug.

</details>

---

### Q14

```jsx
const useStore = create((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

// Outside React — called before any component mounts
useStore.getState().increment();
useStore.getState().increment();

function Counter() {
  const count = useStore((state) => state.count);
  return <p>{count}</p>;
}
```

#### ❓ What does `Counter` display when it first mounts?

<details>
<summary>✅ Answer</summary>

```txt
2
```

**Explanation:** Zustand stores live outside the React component tree. The store is a JavaScript object with its own state. You can call `useStore.getState()` and mutate the store from anywhere — outside components, in plain functions, in Node.js. The two `increment()` calls run before any component mounts and update the store's count to 2. When `Counter` mounts and subscribes via `useStore`, it reads the current state (count = 2) and renders `2`. This is a key advantage over Context API which requires a Provider to be mounted first.

</details>

---

### Q15

```jsx
const useStore = create((set) => ({
  theme: "light",
  setTheme: (theme) => set({ theme }),
}));

// Component A
function ThemeToggle() {
  const setTheme = useStore((state) => state.setTheme);
  return <button onClick={() => setTheme("dark")}>Dark Mode</button>;
}

// Component B — deeply nested, no props passed
function DeepChild() {
  const theme = useStore((state) => state.theme);
  return <div className={theme}>Content</div>;
}
```

#### ❓ Compare this Zustand approach to achieving the same result with Context API.

<details>
<summary>✅ Answer</summary>

```txt
Zustand: No Provider needed. Any component can read/write directly.
Fine-grained subscriptions: ThemeToggle (uses setTheme) does not re-render
when theme changes. DeepChild only re-renders when theme changes.

Context API equivalent: needs ThemeContext.Provider wrapping the tree,
both ThemeToggle and DeepChild re-render when theme changes,
setTheme must be stabilized with useCallback to avoid re-renders.
```

**Explanation:** Zustand's selector subscription model gives you granular control with zero boilerplate. `ThemeToggle` selects only `setTheme` — a function that Zustand keeps stable (same reference) across renders. `DeepChild` selects only `theme` — re-renders only when the theme value changes. With Context API, you need a provider, and all consumers of a combined `{ theme, setTheme }` value re-render on any change. Zustand is effectively "Context with per-selector subscriptions and no Provider boilerplate."

</details>

---

## 4. State Colocation

### Q16

```jsx
function App() {
  const [inputValue, setInputValue] = useState("");

  return (
    <div>
      <Header />
      <SearchBox value={inputValue} onChange={setInputValue} />
      <SearchResults query={inputValue} />
      <Footer />
    </div>
  );
}
```

#### ❓ Every character typed in `SearchBox` re-renders `Header` and `Footer`. How do you fix this without lifting state?

<details>
<summary>✅ Answer</summary>

```txt
Colocate the state — move inputValue down into a wrapper component
that only contains SearchBox and SearchResults.
Header and Footer are no longer siblings of the state, so they don't re-render.
```

**Explanation:** "Colocating state" means moving state as close to where it is used as possible. `Header` and `Footer` do not use `inputValue`, yet they re-render because they are children of `App` which re-renders on state change. The fix: create a `<Search />` component that owns `inputValue` and renders only `SearchBox` and `SearchResults`. `Header` and `Footer` stay in `App` but are no longer in the re-render subtree.

```jsx
function Search() {
  const [inputValue, setInputValue] = useState("");
  return (
    <>
      <SearchBox value={inputValue} onChange={setInputValue} />
      <SearchResults query={inputValue} />
    </>
  );
}

function App() {
  return (
    <div>
      <Header />
      <Search />    {/* only Search re-renders on input change */}
      <Footer />
    </div>
  );
}
```

</details>

---

### Q17

```jsx
function ParentA() {
  const [sharedData, setSharedData] = useState(null);

  return (
    <>
      <ChildA data={sharedData} setData={setSharedData} />
      <ChildB data={sharedData} />
    </>
  );
}
```

#### ❓ Is this a good use of lifting state? When should you instead use Context?

<details>
<summary>✅ Answer</summary>

```txt
This is correct lifting state — appropriate when two siblings share state
and their common parent is close in the tree.

Use Context when:
- The shared state needs to pass through 3+ levels of intermediary components
  (prop drilling becomes painful)
- Many unrelated components across the tree need the same data
- You want to avoid passing props through components that don't use them
```

**Explanation:** Lifting state to the nearest common ancestor is the React-recommended approach for sharing state between siblings. It is simple and explicit. However, if `ChildA` and `ChildB` are deeply nested within a large tree, passing `sharedData` down through many intermediate components that don't use it (prop drilling) becomes cumbersome. Context solves this by making the data available without threading it through every layer. The rule: lift state when the distance is 1–2 levels; use Context when it spans many levels or many unrelated components need access.

</details>

---

### Q18

```jsx
// Option A: Two separate state variables
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");

// Option B: One state object
const [name, setName] = useState({ firstName: "", lastName: "" });
```

#### ❓ Which option is better and when? What is a common mistake with Option B?

<details>
<summary>✅ Answer</summary>

```txt
Option A is better when the fields are truly independent (change separately).
Option B is better when the fields are always updated together (e.g., reset both at once).

Common mistake with Option B: forgetting to spread the existing state:
setName({ firstName: "Alice" })  // BUG: loses lastName
setName(prev => ({ ...prev, firstName: "Alice" }))  // CORRECT
```

**Explanation:** Grouping related state (fields that always change together, like `{ x, y }` coordinates) into an object reduces the number of state variables and simplifies reset logic. But if the fields change independently, separate `useState` calls are cleaner. The critical mistake with object state: `useState`'s setter does NOT shallowly merge (unlike `this.setState` in class components). If you call `setName({ firstName: "Alice" })`, React replaces the entire state with `{ firstName: "Alice" }` and `lastName` is lost.

</details>

---

### Q19

```jsx
function UserProfile({ userId }) {
  const [userData, setUserData] = useState(null);

  useEffect(() => {
    fetchUser(userId).then(setUserData);
  }, [userId]);

  const fullName = `${userData?.firstName} ${userData?.lastName}`;
  const initials = fullName.split(" ").map(n => n[0]).join("");

  // ...
}
```

#### ❓ Should `fullName` and `initials` be state variables or derived values? What is the anti-pattern called?

<details>
<summary>✅ Answer</summary>

```txt
They should be derived values (computed inline), NOT state variables.
Storing them as state is the "redundant state" or "derived state" anti-pattern.
```

**Explanation:** `fullName` and `initials` are deterministically computed from `userData`. There is no independent reason to store them in state — they will always match their derivation from `userData`. If you store them as state, you must keep them in sync with `userData` manually (using `useEffect`), which leads to bugs and extra renders. Compute them inline: `const fullName = ...`. React will recompute them on every render that uses `userData` — and that is fine because React renders are fast and the computation is cheap.

</details>

---

### Q20

```jsx
function App() {
  const [items, setItems] = useState([
    { id: 1, name: "Apple", selected: false },
    { id: 2, name: "Banana", selected: false },
  ]);
  const [selectedItems, setSelectedItems] = useState([]);

  const toggleItem = (id) => {
    setItems(prev => prev.map(item =>
      item.id === id ? { ...item, selected: !item.selected } : item
    ));
    setSelectedItems(items.filter(item => item.selected));
  };
}
```

#### ❓ Is maintaining `selectedItems` as a separate state correct? What is the bug?

<details>
<summary>✅ Answer</summary>

```txt
No — selectedItems is redundant state derived from items.
Bug: setSelectedItems reads stale `items` (before the update from setItems).
selectedItems will always be one step behind.
```

**Explanation:** `setItems(...)` schedules an update — `items` still holds the old value in the current render. The subsequent `setSelectedItems(items.filter(...))` reads the old `items` array, not the updated one. `selectedItems` is always stale by one click. The fix: compute `selectedItems` as a derived value: `const selectedItems = items.filter(item => item.selected)`. This is always consistent with `items` without any synchronization code.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function PriceDisplay({ productId }) {
  const [price, setPrice] = useState(null);
  const [discountedPrice, setDiscountedPrice] = useState(null);
  const DISCOUNT = 0.1;

  useEffect(() => {
    fetchPrice(productId).then(p => {
      setPrice(p);
      setDiscountedPrice(p * (1 - DISCOUNT));
    });
  }, [productId]);

  return <p>Price: {price}, Sale: {discountedPrice}</p>;
}
```

#### ❓ What is wrong with this approach? How should it be rewritten?

<details>
<summary>✅ Answer</summary>

```txt
discountedPrice is redundant state — it is always derived from price.
Storing it separately risks them going out of sync.

Correct approach: derive discountedPrice from price inline.
const discountedPrice = price !== null ? price * (1 - DISCOUNT) : null;
```

**Explanation:** Whenever you can compute a value from existing state, do not store it as a separate state variable. Here, `discountedPrice` is always `price * 0.9`. If you need to update them together, you can store just `price` and compute `discountedPrice` as a constant. Derived state via `useEffect` is especially error-prone because: (1) it adds an extra render cycle, (2) there is a moment between `setPrice` and `setDiscountedPrice` where the two are inconsistent, (3) any missed dependency will cause staleness bugs.

</details>

---

### Q22

```jsx
function Tabs() {
  const [activeTab, setActiveTab] = useState(0);
  const tabs = ["Home", "Profile", "Settings"];

  return (
    <div>
      {tabs.map((tab, i) => (
        <button
          key={i}
          onClick={() => setActiveTab(i)}
          style={{ fontWeight: activeTab === i ? "bold" : "normal" }}
        >
          {tab}
        </button>
      ))}
      <TabContent tab={activeTab} />
    </div>
  );
}

const TabContent = React.memo(({ tab }) => {
  console.log("TabContent renders:", tab);
  return <p>Content for tab {tab}</p>;
});
```

#### ❓ Clicking the already-active tab — does `TabContent` re-render?

<details>
<summary>✅ Answer</summary>

```txt
No — TabContent does NOT re-render.
```

**Explanation:** Clicking the already-active tab calls `setActiveTab(i)` where `i` equals the current `activeTab`. React checks: is the new state (`i`) the same as the old state (`i`)? Yes — same number, same reference. React bails out of the re-render (Object.is comparison). `Tabs` itself does not re-render, so `TabContent` (which is `React.memo`'d anyway) also does not re-render. Even without `React.memo`, React would bail out of updating children if the parent did not re-render.

</details>

---

### Q23

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleOptimisticUpdate = () => {
    // Optimistic: show +1 immediately
    setCount(c => c + 1);

    fetch("/api/increment")
      .then(res => res.json())
      .then(data => setCount(data.count))   // sync with server
      .catch(() => setCount(c => c - 1));   // rollback on error
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleOptimisticUpdate}>+</button>
    </div>
  );
}
```

#### ❓ The user clicks the button 3 times rapidly before the first request completes. What count does the user see immediately?

<details>
<summary>✅ Answer</summary>

```txt
3 (optimistic: each click immediately shows +1)

But after the first server response, the count is set to server's value.
If the server returns 1 (only counted one click), count snaps back to 1
(or whatever the server's count is for the last resolved request).
```

**Explanation:** Each click immediately calls `setCount(c => c + 1)`, showing 1, then 2, then 3 immediately. This is the optimistic update. However, three separate `fetch` requests are flying. When they resolve (possibly out of order), each calls `setCount(data.count)` with the server's count. If requests resolve as 1, 2, 3, the final count is 3 and everything is fine. If the last request resolves as 1 (out-of-order), count snaps back to 1. For robust optimistic updates, track pending requests and only apply the response of the latest request.

</details>

---

### Q24

```jsx
// Local component state
function SearchResults() {
  const [results, setResults] = useState([]);
  const [query, setQuery] = useState("");

  useEffect(() => {
    if (!query) return;
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(data => setResults(data));
  }, [query]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>{results.map(r => <li key={r.id}>{r.title}</li>)}</ul>
    </div>
  );
}
```

#### ❓ The user types `"react"` quickly (5 keystrokes). How many requests are fired? What is a potential bug?

<details>
<summary>✅ Answer</summary>

```txt
5 requests are fired (one per character: "r", "re", "rea", "reac", "react").

Bug: race condition. If the response for "reac" arrives after "react",
the results will show data for "reac" even though the input shows "react".
```

**Explanation:** Each keystroke updates `query`, which triggers the `useEffect`. Five effects fire five fetch requests. Since HTTP responses can arrive out of order (due to network latency), a slower earlier request can override a faster later one. Fix options: (1) debounce the query update (delay firing the effect until the user stops typing); (2) cancel previous requests using an AbortController in the effect cleanup; (3) use a library like React Query or SWR that handles this automatically.

</details>

---

### Q25

```jsx
const useCartStore = create((set, get) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter(i => i.id !== id),
  })),
  total: () => get().items.reduce((sum, i) => sum + i.price, 0),
}));

function CartTotal() {
  const total = useCartStore((state) => state.total());
  console.log("CartTotal renders");
  return <p>Total: ${total}</p>;
}
```

#### ❓ An item's price does not change, but `addItem` is called to add a new free item (price 0). Does `CartTotal` re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — CartTotal re-renders.
The total number is the same (adding 0), but the function call in the selector
creates a potential re-render.
```

**Explanation:** The selector is `state => state.total()`. Every time the store state changes, Zustand calls the selector and compares the result with `Object.is`. After `addItem`, the state object changes (new `items` array). The selector calls `state.total()` which runs `reduce` and returns the same number (since the new item has price 0). `Object.is(oldTotal, newTotal)` → `0 === 0` is `true`, so re-render is prevented. Actually no re-render occurs in this specific case because the computed number is the same. The key takeaway: Zustand compares selector return values with `Object.is` — as long as the value is primitively equal, no re-render fires.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Context API Performance | Q1–Q5 | All-or-nothing subscription, object reference instability, useMemo on context, React.memo + useContext, split contexts |
| Redux / Redux Toolkit | Q6–Q10 | Immer mutation pattern, reassigning vs mutating, selector reference stability, shallowEqual, synchronous dispatch |
| Zustand | Q11–Q15 | Selector subscriptions, Object.is comparison, custom equality, external store access, Zustand vs Context |
| State Colocation | Q16–Q20 | Colocating state, lifting state vs Context, object state spreading, derived vs redundant state |
| Edge Cases | Q21–Q25 | Derived state anti-pattern, tabs bail-out, optimistic updates race conditions, search debounce/abort, Zustand primitive equality |
