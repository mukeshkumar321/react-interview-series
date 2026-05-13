# React State Management

## Table of Contents

1. [What is State Management](#1-what-is-state-management)
2. [Local State with useState](#2-local-state-with-usestate)
3. [Lifting State Up](#3-lifting-state-up)
4. [Context API for State Management](#4-context-api-for-state-management)
5. [useReducer + Context](#5-usereducer--context)
6. [Redux Toolkit](#6-redux-toolkit)
7. [Zustand](#7-zustand)
8. [Jotai / Recoil](#8-jotai--recoil)
9. [When to Use What](#9-when-to-use-what)
10. [Redux Toolkit Deep Dive](#10-redux-toolkit-deep-dive)
11. [Zustand Deep Dive](#11-zustand-deep-dive)
12. [Server State vs Client State](#12-server-state-vs-client-state)
13. [State Normalization](#13-state-normalization)
14. [Common Mistakes](#14-common-mistakes)
15. [Best Practices](#15-best-practices)

---

## 1. What is State Management

State is any data that changes over time and must cause the UI to update when it changes. State management is the pattern you use to store, update, and share that data across your application.

### Local State vs Global State

**Local state** lives inside a single component. It is private to that component and its descendants (unless explicitly passed down). It is destroyed when the component unmounts.

**Global state** is accessible by many components across the component tree, regardless of their position. It persists independently of any individual component's lifecycle.

```txt
Local State
  Component A
    ├── state: { count: 0 }      ← only A and its children know about this
    └── Component B (child of A)
          └── receives count as prop

Global State
  Store / Context
    ├── state: { user: {...}, theme: 'dark' }
    Component A ──── reads user
    Component B ──── reads theme
    Component C ──── reads both
```

### When Local State is Enough

Local state is sufficient when:

- The data is used exclusively within a single component (e.g., a modal's `isOpen` flag)
- The data does not need to be shared with sibling or distant components
- The component is self-contained and reusable (e.g., a custom dropdown managing its own open/closed state)
- The data can be recomputed from props (derived state — do not store it in state at all)

```jsx
// Good: local state for UI-only concern
function Modal({ onClose }) {
  const [isAnimatingOut, setIsAnimatingOut] = useState(false);

  const handleClose = () => {
    setIsAnimatingOut(true);
    setTimeout(() => {
      setIsAnimatingOut(false);
      onClose();
    }, 300);
  };

  return (
    <div className={isAnimatingOut ? 'modal--out' : 'modal'}>
      <button onClick={handleClose}>Close</button>
    </div>
  );
}
```

### When You Need Global State Management

You need global state when:

- Multiple unrelated components need the same piece of data (e.g., current authenticated user)
- Data is updated in one part of the tree and read in a completely different branch
- You want persistence across navigations without prop drilling through many layers
- You need DevTools support for debugging complex state transitions (Redux)

---

## 2. Local State with useState

`useState` is the most fundamental state primitive in React. It is synchronous in scheduling (the state update is scheduled, not applied immediately) and batched in React 18+ for all event handlers and async code.

### Best Use Cases

| Scenario | Example |
|---|---|
| Toggle flags | `isOpen`, `isLoading`, `isEditing` |
| Form field values | `email`, `password`, `searchQuery` |
| UI-only state | `activeTab`, `selectedIndex` |
| Component-level counters | pagination page, retry count |

### Internal Mechanics

When `setState` is called, React enqueues the update, marks the fiber as needing reconciliation, and schedules a re-render. The component function runs again with the new state value.

```jsx
const [count, setCount] = useState(0);

// Functional update form — always use this when new state depends on old state
setCount(prev => prev + 1); // ✅ Safe even in async code

// Direct update — risky when called multiple times in the same event
setCount(count + 1); // ❌ Stale closure in batched updates
```

### Lazy Initialization

If the initial state requires an expensive computation, pass a function to `useState`. The function runs only on the first render.

```jsx
// ❌ Runs expensiveCompute() on every render
const [data, setData] = useState(expensiveCompute());

// ✅ Runs expensiveCompute() only on first render
const [data, setData] = useState(() => expensiveCompute());
```

### Object State

React does not merge object updates automatically with `useState` (unlike `this.setState` in class components). You must spread previous state explicitly.

```jsx
const [form, setForm] = useState({ name: '', email: '' });

// ❌ Replaces the entire object — email is lost
setForm({ name: 'Alice' });

// ✅ Merges correctly
setForm(prev => ({ ...prev, name: 'Alice' }));
```

### Array State

```jsx
const [items, setItems] = useState([]);

// ❌ Mutation — does not trigger re-render
items.push('new item');
setItems(items);

// ✅ New array reference triggers re-render
setItems(prev => [...prev, 'new item']);

// ✅ Removing an item
setItems(prev => prev.filter(item => item.id !== targetId));

// ✅ Updating an item
setItems(prev =>
  prev.map(item => item.id === targetId ? { ...item, done: true } : item)
);
```

---

## 3. Lifting State Up

When two sibling components need to share the same piece of state, the state must be moved (lifted) to their closest common ancestor.

### The Pattern

```txt
Before Lifting:
  Parent
    ├── ComponentA (owns state: selectedId)
    └── ComponentB (needs selectedId — cannot access it)

After Lifting:
  Parent (owns state: selectedId)
    ├── ComponentA (receives selectedId + setter via props)
    └── ComponentB (receives selectedId via props)
```

```jsx
// Before: state siloed in ComponentA
function ComponentA() {
  const [selectedId, setSelectedId] = useState(null);
  return <List onSelect={setSelectedId} />;
}

function ComponentB() {
  // ❌ Cannot access selectedId
}

// After: state lifted to Parent
function Parent() {
  const [selectedId, setSelectedId] = useState(null);
  return (
    <>
      <ComponentA onSelect={setSelectedId} />
      <ComponentB selectedId={selectedId} />
    </>
  );
}
```

### Prop Drilling as a Trade-off

Lifting state up introduces **prop drilling** — passing props through intermediate components that do not themselves use the prop. This is acceptable for 2–3 levels but becomes a maintenance burden beyond that.

```txt
App (owns user)
  └── Layout (passes user down — does NOT use user)
        └── Sidebar (passes user down — does NOT use user)
              └── UserAvatar (finally uses user)
```

At this depth, consider Context API or a state manager instead.

### Controlled vs Uncontrolled Components

Lifting state also applies to form elements. A **controlled component** has its value driven by React state (lifted to the parent), while an **uncontrolled component** manages its own internal DOM state.

```jsx
// Controlled — parent owns the value
function ControlledInput({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}

// Uncontrolled — DOM owns the value
function UncontrolledInput() {
  const ref = useRef(null);
  const handleSubmit = () => console.log(ref.current.value);
  return <input ref={ref} defaultValue="" />;
}
```

---

## 4. Context API for State Management

React Context provides a way to pass data through the component tree without having to pass props at every level. It is part of React's built-in API.

### How Context Works

```jsx
// 1. Create
const ThemeContext = createContext('light'); // default value

// 2. Provide
function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Layout />
    </ThemeContext.Provider>
  );
}

// 3. Consume
function Button() {
  const { theme } = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}
```

### Re-render Behavior

Every component that calls `useContext(SomeContext)` will re-render whenever the context value changes. This is the key limitation of Context for high-frequency updates.

```jsx
// ❌ Every consumer re-renders on every App render because the object reference changes
function App() {
  const [user, setUser] = useState(null);
  return (
    <UserContext.Provider value={{ user, setUser }}>
      {/* All consumers re-render whenever App re-renders */}
    </UserContext.Provider>
  );
}

// ✅ Memoize the value to prevent unnecessary re-renders
function App() {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);
  return (
    <UserContext.Provider value={value}>
      {/* Consumers only re-render when user changes */}
    </UserContext.Provider>
  );
}
```

### Context Default Value

The default value passed to `createContext` is only used when a component consumes the context without a matching Provider above it in the tree. It is most useful for testing components in isolation.

```jsx
const UserContext = createContext({ name: 'Guest', role: 'viewer' });

// This component renders with { name: 'Guest', role: 'viewer' } if no Provider wraps it
function UserBadge() {
  const user = useContext(UserContext);
  return <span>{user.name}</span>;
}
```

### Best Use Cases for Context

| Use Case | Why Context is Good |
|---|---|
| Theme (light/dark) | Changes infrequently, read by many |
| Authenticated user | Stable after login, consumed everywhere |
| Locale / language | Global, changes rarely |
| Feature flags | Read-only configuration |

### Context is NOT a State Manager

Context is a **dependency injection** mechanism, not a state management library. It does not provide:
- Memoized selectors
- Middleware for side effects
- DevTools integration
- Fine-grained subscription (all-or-nothing re-renders)

---

## 5. useReducer + Context

Combining `useReducer` with Context gives you a Redux-like architecture without any external library. It is appropriate for medium-complexity state with multiple related updates.

### Pattern

```jsx
// reducer.js
const initialState = { count: 0, user: null, loading: false };

function appReducer(state, action) {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'SET_USER':
      return { ...state, user: action.payload };
    case 'SET_LOADING':
      return { ...state, loading: action.payload };
    default:
      return state;
  }
}

// context.js
const StateContext = createContext(null);
const DispatchContext = createContext(null);

export function AppProvider({ children }) {
  const [state, dispatch] = useReducer(appReducer, initialState);
  return (
    <StateContext.Provider value={state}>
      <DispatchContext.Provider value={dispatch}>
        {children}
      </DispatchContext.Provider>
    </StateContext.Provider>
  );
}

// Split contexts — components that only dispatch don't re-render when state changes
export const useAppState = () => useContext(StateContext);
export const useAppDispatch = () => useContext(DispatchContext);
```

### Why Split State and Dispatch into Separate Contexts

If you put `{ state, dispatch }` in a single context, every component calling `dispatch`-only (like a button that only fires actions) will re-render whenever state changes. Separating them avoids this.

```jsx
// Component that only dispatches — never re-renders due to state changes
function AddButton() {
  const dispatch = useAppDispatch(); // ✅ stable reference
  return <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>;
}
```

### useReducer vs useState

| Aspect | useState | useReducer |
|---|---|---|
| Complexity | Simple, single values | Complex, multi-field updates |
| Action model | Direct setter | Named actions with payloads |
| Testability | Moderate | High — reducer is a pure function |
| Colocation | In component | Extractable to separate file |
| Best for | Independent state values | Interdependent state transitions |

### When useReducer + Context Reaches Its Limits

- No DevTools (no time-travel debugging)
- No middleware for async actions
- No optimized subscriptions (entire state tree consumed)
- Becomes complex for large applications with many action types

---

## 6. Redux Toolkit

Redux Toolkit (RTK) is the official, opinionated, batteries-included toolset for Redux. It eliminates the verbosity of plain Redux while keeping its predictable, centralized state model.

### Core Concepts

**Single Source of Truth**: The entire application state lives in one store object.

**State is Read-Only**: The only way to change state is to dispatch an action — a plain object describing what happened.

**Reducers are Pure Functions**: Reducers take (state, action) and return a new state without side effects.

```txt
Action Dispatched
      ↓
  Middleware (Thunk, etc.)
      ↓
    Reducer
      ↓
  New State
      ↓
 React Re-renders
```

### Why RTK Over Plain Redux

| Feature | Plain Redux | Redux Toolkit |
|---|---|---|
| Boilerplate | Very high | Minimal |
| Immer integration | Manual (must not mutate) | Built-in (can write "mutating" code) |
| Action creators | Must write manually | Auto-generated by `createSlice` |
| Async | `redux-thunk` manually configured | `createAsyncThunk` built-in |
| Data fetching | Manual | RTK Query |
| DevTools | Manual setup | Enabled by default |

### createSlice

`createSlice` is the central RTK API. It accepts an initial state, a name, and a map of reducer functions. It auto-generates action creators and action types.

```jsx
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0, status: 'idle' },
  reducers: {
    increment(state) {
      state.value += 1; // Immer makes this safe — this is NOT a mutation
    },
    decrement(state) {
      state.value -= 1;
    },
    incrementByAmount(state, action) {
      state.value += action.payload;
    },
    reset(state) {
      state.value = 0;
    },
  },
});

export const { increment, decrement, incrementByAmount, reset } = counterSlice.actions;
export default counterSlice.reducer;
```

### configureStore

```jsx
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './counterSlice';
import userReducer from './userSlice';

const store = configureStore({
  reducer: {
    counter: counterReducer,
    user: userReducer,
  },
  // Redux DevTools Extension enabled by default in development
  // redux-thunk middleware added by default
});

export default store;
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### useSelector and useDispatch

```jsx
import { useSelector, useDispatch } from 'react-redux';
import { increment, decrement, incrementByAmount } from './counterSlice';

function Counter() {
  // useSelector subscribes to the store and triggers re-render only when selected value changes
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(decrement())}>-</button>
      <button onClick={() => dispatch(incrementByAmount(5))}>+5</button>
    </div>
  );
}
```

### createAsyncThunk

`createAsyncThunk` handles the three states of an async operation: `pending`, `fulfilled`, `rejected`. It dispatches lifecycle actions automatically.

```jsx
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit';

export const fetchUser = createAsyncThunk(
  'user/fetchById',  // action type prefix
  async (userId, thunkAPI) => {
    const response = await fetch(`/api/users/${userId}`);
    if (!response.ok) {
      return thunkAPI.rejectWithValue('Failed to fetch user');
    }
    return response.json(); // returned value becomes action.payload in fulfilled
  }
);

const userSlice = createSlice({
  name: 'user',
  initialState: { data: null, loading: false, error: null },
  reducers: {},
  extraReducers: builder => {
    builder
      .addCase(fetchUser.pending, state => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.loading = false;
        state.data = action.payload;
      })
      .addCase(fetchUser.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload;
      });
  },
});
```

### RTK Query

RTK Query is a powerful data fetching and caching layer built on top of Redux Toolkit. It eliminates the need to write `createAsyncThunk` for standard CRUD operations.

```jsx
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const postsApi = createApi({
  reducerPath: 'postsApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Post'],
  endpoints: builder => ({
    getPosts: builder.query({
      query: () => '/posts',
      providesTags: ['Post'],
    }),
    getPostById: builder.query({
      query: id => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Post', id }],
    }),
    createPost: builder.mutation({
      query: body => ({ url: '/posts', method: 'POST', body }),
      invalidatesTags: ['Post'], // automatically refetches getPosts
    }),
    deletePost: builder.mutation({
      query: id => ({ url: `/posts/${id}`, method: 'DELETE' }),
      invalidatesTags: (result, error, id) => [{ type: 'Post', id }],
    }),
  }),
});

export const {
  useGetPostsQuery,
  useGetPostByIdQuery,
  useCreatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

```jsx
// In a component
function PostsList() {
  const { data: posts, isLoading, isError } = useGetPostsQuery();
  const [createPost, { isLoading: isCreating }] = useCreatePostMutation();

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage />;

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

---

## 7. Zustand

Zustand is a lightweight, minimalist state management library for React. It uses a hook-based API with no Provider wrapping requirement and produces almost zero boilerplate.

### Core Concepts

- A **store** is a hook created by the `create()` function
- The store holds both state and actions (unlike Redux which separates them)
- No Provider is needed — the store is module-level
- Subscriptions are selector-based — components only re-render when the subscribed slice changes

### Basic Store

```jsx
import { create } from 'zustand';

const useCounterStore = create((set) => ({
  count: 0,
  increment: () => set(state => ({ count: state.count + 1 })),
  decrement: () => set(state => ({ count: state.count - 1 })),
  incrementBy: (amount) => set(state => ({ count: state.count + amount })),
  reset: () => set({ count: 0 }),
}));

// Usage in component
function Counter() {
  const count = useCounterStore(state => state.count);
  const increment = useCounterStore(state => state.increment);

  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

### No Provider Wrapping

```jsx
// ✅ No Provider needed
function App() {
  return (
    <div>
      <Counter />
      <OtherComponent />
    </div>
  );
}

// Compare to Redux / Context
// ❌ Redux requires Provider
function App() {
  return (
    <Provider store={store}>
      <Counter />
    </Provider>
  );
}
```

### Selectors for Performance

Selecting the entire store causes re-renders on any state change. Use selectors to subscribe only to the slice you need.

```jsx
// ❌ Re-renders on any store change
const state = useCounterStore();

// ✅ Re-renders only when count changes
const count = useCounterStore(state => state.count);

// ✅ Re-renders only when name changes
const name = useUserStore(state => state.name);
```

### Comparison with Redux Toolkit

| Feature | Redux Toolkit | Zustand |
|---|---|---|
| Boilerplate | Low (createSlice) | Minimal |
| Provider required | Yes | No |
| DevTools | Yes (built-in) | Via middleware |
| Immer | Built-in | Via `immer` middleware |
| Async | createAsyncThunk | Direct async functions in store |
| Data fetching | RTK Query | External (React Query) |
| TypeScript | Good | Excellent |
| Bundle size | ~40KB | ~8KB |
| Learning curve | Moderate | Very low |
| Best for | Large/complex apps | Simple to medium apps |

---

## 8. Jotai / Recoil

Both Jotai and Recoil implement the **atomic state model** — state is split into small independent units called atoms rather than one central store.

### Jotai

Atoms are the primitive unit. Components subscribe to specific atoms. Re-renders happen only when subscribed atoms change.

```jsx
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// Define atoms
const countAtom = atom(0);
const nameAtom = atom('Alice');

// Derived atom (computed from other atoms)
const doubleCountAtom = atom(get => get(countAtom) * 2);

// Async atom
const userAtom = atom(async (get) => {
  const id = get(userIdAtom);
  const response = await fetch(`/api/users/${id}`);
  return response.json();
});

function Counter() {
  const [count, setCount] = useAtom(countAtom);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

function DoubleDisplay() {
  const doubleCount = useAtomValue(doubleCountAtom); // read-only
  return <p>{doubleCount}</p>;
}

function NameSetter() {
  const setName = useSetAtom(nameAtom); // write-only — no re-render on name change
  return <button onClick={() => setName('Bob')}>Change Name</button>;
}
```

### Jotai vs Zustand vs Redux

| Feature | Jotai | Zustand | Redux Toolkit |
|---|---|---|---|
| Model | Atomic | Single store | Single store |
| Re-render granularity | Per atom | Per selector | Per selector |
| Boilerplate | Minimal | Minimal | Low |
| Provider | Optional (for SSR scope) | Not needed | Required |
| Async | async atoms | async actions | createAsyncThunk |

### Recoil

Recoil (Meta/Facebook) was the original atomic state library for React. It requires a `RecoilRoot` Provider and uses `atom()` and `selector()` primitives. Jotai is often preferred today due to its simpler API and no vendor lock-in.

---

## 9. When to Use What

### Decision Tree

```text
Does the state need to be shared
between multiple components?
        ↓
       No → useState (local state)
        ↓
       Yes
        ↓
Is it only 2–3 levels of prop
drilling? (e.g., parent → child → grandchild)
        ↓
    Yes → Lift state up (useState in common ancestor)
        ↓
       No
        ↓
Is it low-frequency, cross-cutting
concern? (theme, auth, locale)
        ↓
    Yes → Context API
        ↓
       No
        ↓
Do you need DevTools, strict
unidirectional flow, or is the team
already using Redux?
        ↓
    Yes → Redux Toolkit
        ↓
       No
        ↓
    Zustand (simple, modern, minimal)
```

### Detailed Comparison Table

| Dimension | Local State (useState) | Context API | Redux Toolkit | Zustand |
|---|---|---|---|---|
| Boilerplate | None | Low | Low (RTK) | Minimal |
| DevTools | No | No | Yes | Yes (via middleware) |
| Async handling | Manual | Manual | createAsyncThunk / RTK Query | Direct async in store |
| Re-render optimization | Manual (memo) | Hard (all consumers) | useSelector | Selector-based |
| Provider required | No | Yes | Yes | No |
| Bundle size | 0 (built-in) | 0 (built-in) | ~40KB | ~8KB |
| Learning curve | None | Low | Moderate | Very low |
| Best use case | Component-specific UI | Shared infrequent data | Complex global state | Simple global state |
| Server state | Not suited | Not suited | RTK Query | Use React Query |

---

## 10. Redux Toolkit Deep Dive

### Slice Anatomy

A slice encapsulates a single domain of your state. It contains:

```jsx
const mySlice = createSlice({
  name: 'myFeature',          // prefix for action types: 'myFeature/actionName'
  initialState,               // the initial state value
  reducers: {                 // synchronous reducers
    actionA(state, action) { /* Immer-safe mutations */ },
  },
  extraReducers: builder => { // handles actions from other slices or thunks
    builder.addCase(someThunk.fulfilled, (state, action) => {});
  },
  selectors: {                // co-located selectors (RTK 2.0+)
    selectValue: state => state.value,
  },
});
```

### Actions Auto-Generated from Slice

RTK's `createSlice` generates action creators matching the reducer keys. The action type string is `"<name>/<reducerKey>"`.

```jsx
const { increment, decrement } = counterSlice.actions;

console.log(increment());
// { type: 'counter/increment', payload: undefined }

console.log(incrementByAmount(10));
// { type: 'counter/incrementByAmount', payload: 10 }
```

### Async Thunks — Lifecycle Actions

```txt
dispatch(fetchUser(42))
      ↓
fetchUser.pending dispatched   → { type: 'user/fetchById/pending' }
      ↓
  (await fetch resolves)
      ↓
fetchUser.fulfilled dispatched → { type: 'user/fetchById/fulfilled', payload: userData }
                OR
fetchUser.rejected dispatched  → { type: 'user/fetchById/rejected', payload: errorMsg }
```

### Extra Reducers

`extraReducers` handles action types not generated by this slice — typically from `createAsyncThunk` or other slices.

```jsx
extraReducers: builder => {
  // builder pattern is type-safe
  builder
    .addCase(fetchUser.pending, (state) => {
      state.status = 'loading';
    })
    .addCase(fetchUser.fulfilled, (state, action) => {
      state.status = 'succeeded';
      state.entities[action.payload.id] = action.payload;
    })
    .addCase(fetchUser.rejected, (state, action) => {
      state.status = 'failed';
      state.error = action.error.message;
    })
    .addMatcher(
      action => action.type.endsWith('/pending'),
      state => { state.globalLoading = true; }
    );
},
```

### Middleware

RTK's `configureStore` comes with `redux-thunk` pre-installed. You can extend with custom middleware:

```jsx
import { configureStore } from '@reduxjs/toolkit';
import { loggerMiddleware } from './middleware';

const store = configureStore({
  reducer: rootReducer,
  middleware: getDefaultMiddleware =>
    getDefaultMiddleware().concat(loggerMiddleware),
});
```

Custom middleware signature (curried function):

```jsx
const loggerMiddleware = storeAPI => next => action => {
  console.log('dispatching', action);
  const result = next(action);
  console.log('next state', storeAPI.getState());
  return result;
};
```

### Immer Integration

RTK uses Immer under the hood in `createSlice` reducers. This means you can write code that appears to mutate state, and Immer converts it to an immutable update.

```jsx
// ✅ Inside createSlice reducer — Immer handles this
increment(state) {
  state.value += 1; // Immer creates new state, does NOT mutate
}

// ❌ Outside createSlice — no Immer, this mutates the actual object
const state = store.getState();
state.counter.value += 1; // NEVER do this
```

### Selector Patterns with createSelector

```jsx
import { createSelector } from '@reduxjs/toolkit';

// Basic selector
const selectUsers = state => state.users.entities;
const selectCurrentUserId = state => state.auth.currentUserId;

// Memoized derived selector
export const selectCurrentUser = createSelector(
  [selectUsers, selectCurrentUserId],
  (users, currentUserId) => users[currentUserId]
);

// Parameterized selector factory
export const makeSelectPostsByUser = (userId) =>
  createSelector(
    state => state.posts.entities,
    posts => Object.values(posts).filter(p => p.authorId === userId)
  );
```

---

## 11. Zustand Deep Dive

### Store Creation with create()

```jsx
import { create } from 'zustand';

interface UserState {
  user: User | null;
  isAuthenticated: boolean;
  login: (userData: User) => void;
  logout: () => void;
  updateName: (newName: string) => void;
}

const useUserStore = create<UserState>((set, get) => ({
  user: null,
  isAuthenticated: false,

  login: (userData) => set({ user: userData, isAuthenticated: true }),

  logout: () => set({ user: null, isAuthenticated: false }),

  // get() accesses current state inside actions
  updateName: (newName) => set(state => ({
    user: state.user ? { ...state.user, name: newName } : null,
  })),
}));
```

### Accessing State with Selector

```jsx
// Granular subscription — component only re-renders when user.name changes
const userName = useUserStore(state => state.user?.name);

// Multiple values with shallow equality check
import { shallow } from 'zustand/shallow';

const { user, isAuthenticated } = useUserStore(
  state => ({ user: state.user, isAuthenticated: state.isAuthenticated }),
  shallow // prevents re-render if both values are shallowly equal to previous
);
```

### Actions Inside Store

Zustand co-locates actions with state. Actions access the latest state via the `set` function's callback form, or via `get()` for reading without triggering re-renders.

```jsx
const useCartStore = create((set, get) => ({
  items: [],
  total: 0,

  addItem: (item) => set(state => ({
    items: [...state.items, item],
    total: state.total + item.price,
  })),

  removeItem: (itemId) => {
    const { items } = get(); // get current state without subscribing
    const item = items.find(i => i.id === itemId);
    if (!item) return;
    set(state => ({
      items: state.items.filter(i => i.id !== itemId),
      total: state.total - item.price,
    }));
  },

  clearCart: () => set({ items: [], total: 0 }),
}));
```

### Persist Middleware

Persist state to localStorage (or any storage engine) automatically:

```jsx
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: 'light',
      language: 'en',
      setTheme: (theme) => set({ theme }),
      setLanguage: (language) => set({ language }),
    }),
    {
      name: 'user-settings',          // localStorage key
      storage: createJSONStorage(() => localStorage),
      partialize: state => ({          // only persist these fields
        theme: state.theme,
        language: state.language,
      }),
    }
  )
);
```

### DevTools Middleware

```jsx
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

const useStore = create(
  devtools(
    (set) => ({
      count: 0,
      increment: () => set(
        state => ({ count: state.count + 1 }),
        false,         // false = merge, true = replace entire state
        'increment'    // action name shown in DevTools
      ),
    }),
    { name: 'CounterStore' }
  )
);
```

### Async Actions in Zustand

```jsx
const usePostsStore = create((set) => ({
  posts: [],
  loading: false,
  error: null,

  fetchPosts: async () => {
    set({ loading: true, error: null });
    try {
      const response = await fetch('/api/posts');
      const posts = await response.json();
      set({ posts, loading: false });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  },
}));
```

### Zustand Outside React

Zustand stores can be accessed and updated outside of React components:

```jsx
// Read state anywhere
const currentUser = useUserStore.getState().user;

// Update state anywhere (e.g., in an interceptor or event listener)
useUserStore.setState({ user: null });

// Subscribe to changes outside React
const unsubscribe = useUserStore.subscribe(
  state => state.user,
  (newUser, prevUser) => {
    console.log('User changed:', newUser);
  }
);
```

---

## 12. Server State vs Client State

This is one of the most important distinctions in modern React architecture.

### Client State

Client state is UI-driven data that does not need to be synchronized with a remote server. It lives entirely in the browser.

| Examples | Library |
|---|---|
| Modal open/closed | useState |
| Selected tab | useState |
| Dark/light theme | Zustand / Context |
| Form draft values | useState / React Hook Form |
| Authentication status (boolean) | Zustand / Redux |

### Server State

Server state is data that originates from a remote source. It has unique challenges:

- **Asynchronous** — must be fetched, not computed
- **Stale** — the cached copy may be outdated
- **Shared** — other clients may change it without your knowledge
- **Cacheable** — avoid refetching if fresh data is already available

| Examples | Library |
|---|---|
| User profile from API | React Query / RTK Query |
| Product list | React Query / RTK Query |
| Paginated search results | React Query / RTK Query |
| Real-time data | SWR / React Query with refetchInterval |

### React Query for Server State

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserProfile({ userId }) {
  const { data: user, isLoading, isError } = useQuery({
    queryKey: ['user', userId],    // cache key
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,     // data fresh for 5 minutes
    gcTime: 10 * 60 * 1000,       // keep in cache for 10 minutes after unmount
  });

  const queryClient = useQueryClient();
  const { mutate: updateUser } = useMutation({
    mutationFn: (updates) => patchUser(userId, updates),
    onSuccess: (data) => {
      // Optimistic update or invalidation
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
  });

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage />;
  return <div>{user.name}</div>;
}
```

### Mixing Both

A well-architected app keeps server state and client state separate:

```txt
Server State (React Query / RTK Query)
  └── posts, users, products (remote data)

Client State (Zustand / Redux)
  └── theme, selectedTab, modalOpen (UI data)

Local State (useState)
  └── inputValue, isHovered (component-level UI)
```

### isFetching vs isLoading

| Property | Meaning |
|---|---|
| `isLoading` | No cached data AND currently fetching |
| `isFetching` | Any fetch in progress (including background refetch) |
| `isStale` | Cached data exists but staleTime has elapsed |
| `isPending` | (React Query v5) Replaces isLoading for clarity |

---

## 13. State Normalization

Normalized state stores entities in a flat key-value lookup (by ID) instead of nested arrays. This avoids duplication and enables O(1) entity lookups.

### Flat vs Nested State

```txt
❌ Nested (denormalized):
{
  posts: [
    { id: 1, title: 'A', author: { id: 10, name: 'Alice' } },
    { id: 2, title: 'B', author: { id: 10, name: 'Alice' } },
  ]
}
Problem: Alice appears in every post. Updating her name requires iterating every post.

✅ Normalized (flat):
{
  posts: {
    ids: [1, 2],
    entities: {
      1: { id: 1, title: 'A', authorId: 10 },
      2: { id: 2, title: 'B', authorId: 10 },
    }
  },
  users: {
    ids: [10],
    entities: {
      10: { id: 10, name: 'Alice' }
    }
  }
}
```

### Entity Adapter in RTK

RTK's `createEntityAdapter` provides a pre-built CRUD state shape with sorting, normalization, and selectors.

```jsx
import { createEntityAdapter, createSlice } from '@reduxjs/toolkit';

const postsAdapter = createEntityAdapter({
  sortComparer: (a, b) => b.date.localeCompare(a.date), // newest first
});

const postsSlice = createSlice({
  name: 'posts',
  initialState: postsAdapter.getInitialState({ status: 'idle' }),
  reducers: {
    postAdded: postsAdapter.addOne,
    postsReceived: postsAdapter.setAll,
    postUpdated: postsAdapter.updateOne,
    postRemoved: postsAdapter.removeOne,
  },
  extraReducers: builder => {
    builder.addCase(fetchPosts.fulfilled, (state, action) => {
      state.status = 'succeeded';
      postsAdapter.setAll(state, action.payload);
    });
  },
});

// Auto-generated selectors
export const {
  selectAll: selectAllPosts,
  selectById: selectPostById,
  selectIds: selectPostIds,
} = postsAdapter.getSelectors(state => state.posts);
```

### Selectors for Derived Data

Use `createSelector` from Reselect (bundled with RTK) to memoize derived computations:

```jsx
import { createSelector } from '@reduxjs/toolkit';

const selectAllPosts = state => state.posts.entities;
const selectCurrentUser = state => state.auth.user;

// Only recomputes when posts or currentUser changes
export const selectCurrentUserPosts = createSelector(
  [selectAllPosts, selectCurrentUser],
  (posts, user) => Object.values(posts).filter(post => post.authorId === user?.id)
);
```

---

## 14. Common Mistakes

### Using Redux for Local UI State

```jsx
// ❌ Anti-pattern: modal visibility in Redux store
dispatch(setModalOpen(true));
// This is overkill. Redux adds indirection with no benefit for local UI state.

// ✅ Use local state
const [isModalOpen, setIsModalOpen] = useState(false);
```

### Not Memoizing Selectors

```jsx
// ❌ Creates a new array reference every render — causes unnecessary re-renders
const activeUsers = useSelector(state =>
  state.users.filter(u => u.isActive)
);

// ✅ Memoized selector — stable reference when inputs don't change
const selectActiveUsers = createSelector(
  state => state.users,
  users => users.filter(u => u.isActive)
);
const activeUsers = useSelector(selectActiveUsers);
```

### Mutating Redux State Outside createSlice

```jsx
// ❌ Direct mutation — bypasses Redux, causes subtle bugs
const state = store.getState();
state.users.push({ id: 99, name: 'Hacker' }); // This breaks Redux contract

// ✅ Dispatch an action
store.dispatch(userAdded({ id: 99, name: 'Alice' }));
```

### Context for High-Frequency Updates

```jsx
// ❌ Using Context for mouse position — re-renders every consumer on every mousemove
const MouseContext = createContext({ x: 0, y: 0 });
function App() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  return (
    <MouseContext.Provider value={pos}>
      <Content onMouseMove={e => setPos({ x: e.clientX, y: e.clientY })} />
    </MouseContext.Provider>
  );
}

// ✅ Use a ref or Zustand with selectors for high-frequency updates
```

### Storing Derived State

```jsx
// ❌ Keeping count in state when it can be derived
const [items, setItems] = useState([]);
const [itemCount, setItemCount] = useState(0); // redundant

// ✅ Derive it
const [items, setItems] = useState([]);
const itemCount = items.length; // computed during render
```

### Stale Closure in useSelector

```jsx
// ❌ Inline function creates new selector on every render (minor perf issue)
const data = useSelector(state => expensiveTransform(state.items));

// ✅ Define selector outside component
const selectTransformedItems = createSelector(
  state => state.items,
  items => expensiveTransform(items)
);
const data = useSelector(selectTransformedItems);
```

---

## 15. Best Practices

### Start Simple, Add Complexity When Needed

```txt
useState
  → When component is the only consumer

Lift State Up
  → When 2–3 closely related components share state

Context API
  → When state is low-frequency and broadly consumed (theme, auth)

Zustand
  → When global state is needed but Redux is overkill

Redux Toolkit
  → When app is large, team is many, strict patterns needed, complex async

React Query / RTK Query
  → Always for server/remote state (never store API data in useState)
```

### Separate Server State from Client State

Never fetch API data and store it in `useState` for non-trivial apps. Use React Query or RTK Query which handle caching, background refetching, pagination, and stale-while-revalidate automatically.

### Normalize Complex Nested Data

If your state has entities with relationships (users → posts → comments), normalize them with RTK's `createEntityAdapter` or manually to avoid duplication and stale nested references.

### Memoize Expensive Selectors

Any selector that filters, maps, or reduces data should be wrapped in `createSelector` (Redux) or computed once outside the component with `useMemo` (React).

### Co-locate State with Its Consumer

```txt
Rule: State should live as close to where it is used as possible.

If only one component uses it → useState in that component
If a feature's components use it → useState or Zustand in that feature
If the whole app uses it → Global store (Redux / Zustand)
```

### Use TypeScript for Store Types

Both Redux Toolkit and Zustand have first-class TypeScript support. Always type your state shape, action payloads, and selectors for refactor safety and IDE autocompletion.

```jsx
// RTK typed hooks
import type { RootState, AppDispatch } from './store';
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
```

### Feature Slices Over Giant Slices

```txt
❌ One giant slice with everything:
  appSlice: { user, posts, comments, ui, settings }

✅ One slice per domain:
  userSlice: { data, status }
  postsSlice: { entities, ids, status }
  uiSlice: { modalOpen, selectedTab }
```

Smaller slices are easier to test, reason about, and maintain. Each slice should own one clear domain of state.

---
