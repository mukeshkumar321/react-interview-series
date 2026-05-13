# Redux Toolkit

## Table of Contents

1. [What is Redux Toolkit](#1-what-is-redux-toolkit)
2. [Core Concepts](#2-core-concepts)
3. [configureStore](#3-configurestore)
4. [createSlice](#4-createslice)
5. [Immer Under the Hood](#5-immer-under-the-hood)
6. [useSelector](#6-useselector)
7. [useDispatch](#7-usedispatch)
8. [createAsyncThunk](#8-createasyncthunk)
9. [RTK Query](#9-rtk-query)
10. [createEntityAdapter](#10-createentityadapter)
11. [Store Structure and Feature-Based Slices](#11-store-structure-and-feature-based-slices)
12. [DevTools Integration](#12-devtools-integration)
13. [RTK vs Context API vs Zustand](#13-rtk-vs-context-api-vs-zustand)
14. [Redux Middleware](#14-redux-middleware)
15. [Common Mistakes](#15-common-mistakes)
16. [Best Practices](#16-best-practices)

---

## 1. What is Redux Toolkit

Redux Toolkit (RTK) is the official, opinionated toolset for Redux development. It was created to address the three most common Redux complaints:

- **Too much boilerplate** — action types, action creators, reducers, separate files
- **Configuration complexity** — setting up a store with middleware, DevTools, enhancers
- **Immutability requirements** — verbose spread syntax to avoid mutating state

```text
Legacy Redux vs Redux Toolkit:

Legacy:                              RTK:
────────────────────────────────     ────────────────────────────────
const ADD_TODO = 'ADD_TODO';         createSlice handles all of this
const addTodo = (todo) => ({                in a single call
  type: ADD_TODO,                    ↓
  payload: todo,
});
function reducer(state = [], action) {
  switch (action.type) {
    case ADD_TODO:
      return [...state, action.payload];
    default:
      return state;
  }
}
```

### What RTK Provides

| Feature | Legacy Redux | Redux Toolkit |
|---|---|---|
| Reducer + actions | Manual switch + action creators | `createSlice` |
| Immutability | Manual spread (`[...arr, item]`) | Immer built-in (write mutations) |
| Async actions | `redux-thunk` separately | `createAsyncThunk` built-in |
| Store setup | Manual `createStore` + middleware | `configureStore` |
| API fetching | External (React Query, SWR) | RTK Query built-in |
| DevTools | Manual enhancer | Enabled by default |
| TypeScript | Verbose type definitions | Excellent inference |

---

## 2. Core Concepts

### The Redux Data Flow

```text
User clicks button
      ↓
dispatch(action)
      ↓
Reducer processes action
(produces new state immutably)
      ↓
Store notifies all subscribers
      ↓
useSelector re-runs selectors
      ↓
Changed components re-render
```

### Terminology

| Term | Definition |
|---|---|
| **Store** | Single source of truth — the object holding the full application state tree |
| **State** | Plain JavaScript object describing the current state of the app |
| **Action** | Plain object with a `type` string and optional `payload` |
| **Action Creator** | Function that returns an action object |
| **Reducer** | Pure function `(state, action) => newState` |
| **Slice** | RTK's bundling of state + reducers + action creators for one feature |
| **Selector** | Function that extracts a piece of state from the store |
| **Thunk** | Function that receives `dispatch` and `getState` — for async logic |
| **Middleware** | Functions that intercept dispatched actions before they reach reducers |

---

## 3. configureStore

`configureStore` replaces Redux's legacy `createStore` and automatically sets up recommended middleware and DevTools.

```ts
// store/store.ts
import { configureStore } from '@reduxjs/toolkit';
import counterReducer from './features/counter/counterSlice';
import postsReducer from './features/posts/postsSlice';
import authReducer from './features/auth/authSlice';

const store = configureStore({
  reducer: {
    counter: counterReducer,
    posts: postsReducer,
    auth: authReducer,
  },

  // ─── Middleware ───────────────────────────────────────────────────────────
  // RTK includes redux-thunk by default. To add more:
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(myCustomMiddleware),

  // ─── DevTools ────────────────────────────────────────────────────────────
  // Enabled in development, disabled in production by default
  devTools: process.env.NODE_ENV !== 'production',

  // ─── Enhancers ───────────────────────────────────────────────────────────
  // Add store enhancers (advanced — rarely needed)
  enhancers: [],
});

export default store;

// TypeScript types for entire app
export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

### Providing the Store

```tsx
// main.tsx
import { Provider } from 'react-redux';
import { store } from './store/store';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <Provider store={store}>
    <App />
  </Provider>
);
```

### TypedHooks (Best Practice)

```ts
// store/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './store';

// Use these typed hooks throughout your app instead of plain useDispatch/useSelector
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

---

## 4. createSlice

`createSlice` is the most important RTK API. It combines initial state, reducers, and auto-generated action creators in one call.

### Basic Counter Slice

```ts
// features/counter/counterSlice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

interface CounterState {
  value: number;
  step: number;
}

const initialState: CounterState = {
  value: 0,
  step: 1,
};

const counterSlice = createSlice({
  name: 'counter',           // Prefix for action types: 'counter/increment'
  initialState,
  reducers: {
    increment(state) {
      state.value += state.step;  // ← Direct mutation (Immer handles immutability)
    },
    decrement(state) {
      state.value -= state.step;
    },
    incrementByAmount(state, action: PayloadAction<number>) {
      state.value += action.payload;
    },
    reset(state) {
      state.value = 0;
    },
    setStep(state, action: PayloadAction<number>) {
      state.step = action.payload;
    },
  },
});

// Auto-generated action creators — one per reducer function
export const { increment, decrement, incrementByAmount, reset, setStep } =
  counterSlice.actions;

// Reducer to add to configureStore
export default counterSlice.reducer;
```

### Auto-Generated Action Types

```ts
// Action creators are auto-generated with the slice name as prefix
increment()         // { type: 'counter/increment' }
decrement()         // { type: 'counter/decrement' }
incrementByAmount(5) // { type: 'counter/incrementByAmount', payload: 5 }
```

### Prepare Callback for Custom Payload

```ts
const todosSlice = createSlice({
  name: 'todos',
  initialState: [] as Todo[],
  reducers: {
    addTodo: {
      reducer(state, action: PayloadAction<Todo>) {
        state.push(action.payload);
      },
      prepare(text: string) {
        // Prepare callback customizes payload before reducer runs
        return {
          payload: {
            id: nanoid(),   // generate ID here, not in reducer
            text,
            completed: false,
          },
        };
      },
    },
  },
});

// Usage: dispatch(addTodo('Buy milk'))
// Not: dispatch(addTodo({ id: '...', text: 'Buy milk', completed: false }))
```

---

## 5. Immer Under the Hood

RTK uses [Immer](https://immerjs.github.io/immer/) inside `createSlice` reducers. Immer wraps the state in a `Proxy` and records mutations. After the reducer runs, Immer applies the mutations to produce a new immutable state object.

### Writing "Mutations" Safely

```ts
reducers: {
  // ✅ These "mutations" are safe inside createSlice — Immer handles them
  addItem(state, action) {
    state.items.push(action.payload);       // push — immutable in Immer
  },
  removeItem(state, action) {
    const idx = state.items.findIndex(i => i.id === action.payload);
    if (idx !== -1) state.items.splice(idx, 1);  // splice — safe in Immer
  },
  updateUser(state, action) {
    const user = state.users.find(u => u.id === action.payload.id);
    if (user) {
      user.name = action.payload.name;  // direct property mutation — safe
    }
  },
  // ✅ Returning a new value also works
  reset() {
    return initialState;  // return replaces state entirely
  },
}
```

### Mutate OR Return — Never Both

```ts
// ❌ Wrong — doing BOTH mutation and return is an Immer error
addItem(state, action) {
  state.items.push(action.payload);
  return state;  // ERROR: cannot both mutate and return in Immer
}

// ✅ Option A: Mutate in place (most common)
addItem(state, action) {
  state.items.push(action.payload);
}

// ✅ Option B: Return entirely new state
addItem(state, action) {
  return { ...state, items: [...state.items, action.payload] };
}
```

### Immer Limits: What Cannot be Mutated

Immer only intercepts mutations on the **proxied state object**. You cannot mutate variables outside the Proxy:

```ts
// ❌ Wrong — mutating a local variable, not the proxied state
removeAll(state) {
  let items = state.items;
  items = [];  // local reassignment, not a mutation of state.items
}

// ✅ Correct
removeAll(state) {
  state.items = [];  // direct mutation of state.items property
}
```

---

## 6. useSelector

`useSelector` subscribes a React component to a slice of the Redux store.

### Basic Usage

```tsx
import { useSelector } from 'react-redux';
import type { RootState } from './store/store';

function CounterDisplay() {
  // Component re-renders when the selected value changes
  const count = useSelector((state: RootState) => state.counter.value);
  return <p>Count: {count}</p>;
}
```

### Selector Memoization with reselect

```ts
import { createSelector } from '@reduxjs/toolkit';

// ─── Input selectors ─────────────────────────────────────────────────────────
const selectItems = (state: RootState) => state.cart.items;
const selectFilter = (state: RootState) => state.cart.filter;

// ─── Memoized output selector ─────────────────────────────────────────────────
const selectFilteredItems = createSelector(
  [selectItems, selectFilter],
  (items, filter) => {
    // This only recomputes when items OR filter changes
    // Not on every re-render
    return items.filter(item => item.category === filter);
  }
);

// Usage in component
const filteredItems = useSelector(selectFilteredItems);
```

### Re-render Behavior

```tsx
// ❌ Wrong — new object reference every selector call → infinite re-renders
const { name, email } = useSelector((state: RootState) => ({
  name: state.user.name,
  email: state.user.email,
}));

// ✅ Option A: Separate selectors
const name = useSelector((state: RootState) => state.user.name);
const email = useSelector((state: RootState) => state.user.email);

// ✅ Option B: shallowEqual for object selectors
import { shallowEqual } from 'react-redux';
const { name, email } = useSelector(
  (state: RootState) => ({ name: state.user.name, email: state.user.email }),
  shallowEqual
);
```

**Default comparison:** `useSelector` uses strict reference equality (`===`) by default. If the selector returns a new object on every call, the component re-renders on every store update.

---

## 7. useDispatch

`useDispatch` returns the Redux store's `dispatch` function for dispatching actions.

```tsx
import { useDispatch } from 'react-redux';
import { increment, incrementByAmount, reset } from './counterSlice';

function CounterControls() {
  const dispatch = useDispatch<AppDispatch>();  // typed dispatch

  return (
    <div>
      <button onClick={() => dispatch(increment())}>+</button>
      <button onClick={() => dispatch(incrementByAmount(5))}>+5</button>
      <button onClick={() => dispatch(reset())}>Reset</button>
    </div>
  );
}
```

### Dispatching Thunks

```tsx
import { fetchUser } from './userSlice';

function UserLoader({ userId }) {
  const dispatch = useDispatch<AppDispatch>();

  useEffect(() => {
    dispatch(fetchUser(userId));  // dispatches a thunk
  }, [userId, dispatch]);
}
```

---

## 8. createAsyncThunk

`createAsyncThunk` creates thunk action creators that handle async logic with automatic pending/fulfilled/rejected lifecycle actions.

### Basic Setup

```ts
// features/users/usersSlice.ts
import { createAsyncThunk, createSlice, PayloadAction } from '@reduxjs/toolkit';

interface User { id: number; name: string; email: string; }
interface UsersState {
  users: User[];
  loading: 'idle' | 'pending' | 'succeeded' | 'failed';
  error: string | null;
}

// ─── Thunk Action Creator ──────────────────────────────────────────────────
export const fetchUsers = createAsyncThunk(
  'users/fetchAll',      // action type prefix
  async (_, thunkAPI) => {
    try {
      const res = await fetch('/api/users');
      if (!res.ok) throw new Error('Server error');
      return await res.json() as User[];  // returned value becomes payload
    } catch (err) {
      // Use rejectWithValue to pass a custom error payload
      return thunkAPI.rejectWithValue((err as Error).message);
    }
  }
);

// ─── Slice with extraReducers ──────────────────────────────────────────────
const usersSlice = createSlice({
  name: 'users',
  initialState: { users: [], loading: 'idle', error: null } as UsersState,
  reducers: {},  // no sync reducers for this example
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => {
        state.loading = 'pending';
        state.error = null;
      })
      .addCase(fetchUsers.fulfilled, (state, action: PayloadAction<User[]>) => {
        state.loading = 'succeeded';
        state.users = action.payload;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.loading = 'failed';
        state.error = action.payload as string;
      });
  },
});

export default usersSlice.reducer;
```

### Thunk Lifecycle Actions

```text
dispatch(fetchUsers())
     ↓
'users/fetchAll/pending' dispatched → state.loading = 'pending'
     ↓
async function runs
     ↓
  ┌────────────────────────────────────────────────────────────────────┐
  │ return data (fulfilled)           │ return rejectWithValue/throw  │
  │ 'users/fetchAll/fulfilled'        │ 'users/fetchAll/rejected'     │
  │ action.payload = returned data    │ action.payload = error value  │
  └────────────────────────────────────────────────────────────────────┘
```

### Accessing thunkAPI

```ts
export const createPost = createAsyncThunk(
  'posts/create',
  async (postData: NewPost, { dispatch, getState, rejectWithValue, signal }) => {
    const state = getState() as RootState;
    const userId = state.auth.user?.id;

    // AbortController integration — signal passed to fetch
    const res = await fetch('/api/posts', {
      method: 'POST',
      body: JSON.stringify({ ...postData, userId }),
      signal,  // automatically abort if dispatch(createPost.abort()) is called
    });

    if (!res.ok) return rejectWithValue('Failed to create post');

    const post = await res.json();
    dispatch(addPostToFeed(post));  // dispatch other actions from within thunk
    return post;
  }
);
```

---

## 9. RTK Query

RTK Query is a powerful data fetching and caching solution built into RTK. It eliminates the need for `createAsyncThunk` for standard CRUD operations.

### API Definition

```ts
// services/postsApi.ts
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

interface Post { id: number; title: string; body: string; }

export const postsApi = createApi({
  reducerPath: 'postsApi',   // key in the Redux store
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Post'],         // cache tags for invalidation

  endpoints: (builder) => ({
    // ─── Query (GET) ────────────────────────────────────────────────────────
    getPosts: builder.query<Post[], void>({
      query: () => '/posts',
      providesTags: ['Post'],
    }),

    getPost: builder.query<Post, number>({
      query: (id) => `/posts/${id}`,
      providesTags: (result, error, id) => [{ type: 'Post', id }],
    }),

    // ─── Mutation (POST/PUT/DELETE) ───────────────────────────────────────
    createPost: builder.mutation<Post, Partial<Post>>({
      query: (newPost) => ({
        url: '/posts',
        method: 'POST',
        body: newPost,
      }),
      invalidatesTags: ['Post'],   // invalidates all Post cache entries
    }),

    updatePost: builder.mutation<Post, { id: number; changes: Partial<Post> }>({
      query: ({ id, changes }) => ({
        url: `/posts/${id}`,
        method: 'PATCH',
        body: changes,
      }),
      invalidatesTags: (result, error, { id }) => [{ type: 'Post', id }],
    }),

    deletePost: builder.mutation<void, number>({
      query: (id) => ({ url: `/posts/${id}`, method: 'DELETE' }),
      invalidatesTags: (result, error, id) => [{ type: 'Post', id }],
    }),
  }),
});

// Auto-generated hooks
export const {
  useGetPostsQuery,
  useGetPostQuery,
  useCreatePostMutation,
  useUpdatePostMutation,
  useDeletePostMutation,
} = postsApi;
```

### Add to Store

```ts
import { postsApi } from './services/postsApi';

const store = configureStore({
  reducer: {
    counter: counterReducer,
    [postsApi.reducerPath]: postsApi.reducer,  // required
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(postsApi.middleware),  // required for caching
});
```

### Using RTK Query Hooks

```tsx
function PostsList() {
  const { data: posts, isLoading, isError, error } = useGetPostsQuery();

  if (isLoading) return <Spinner />;
  if (isError) return <p>Error: {error.toString()}</p>;
  return <ul>{posts?.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}

function CreatePostButton() {
  const [createPost, { isLoading }] = useCreatePostMutation();

  return (
    <button
      disabled={isLoading}
      onClick={() => createPost({ title: 'New Post', body: 'Content...' })}
    >
      Create
    </button>
  );
}
```

---

## 10. createEntityAdapter

`createEntityAdapter` manages normalized state for collections of entities (arrays of objects with a common ID field).

```ts
import { createEntityAdapter, createSlice } from '@reduxjs/toolkit';

interface Todo { id: string; text: string; completed: boolean; }

const todosAdapter = createEntityAdapter<Todo>({
  sortComparer: (a, b) => a.text.localeCompare(b.text),
});

// Initial state: { ids: [], entities: {} }
const initialState = todosAdapter.getInitialState({
  loading: false,
});

const todosSlice = createSlice({
  name: 'todos',
  initialState,
  reducers: {
    // Adapter provides CRUD operations
    addTodo: todosAdapter.addOne,
    addManyTodos: todosAdapter.addMany,
    updateTodo: todosAdapter.updateOne,
    removeTodo: todosAdapter.removeOne,
    removeAllTodos: todosAdapter.removeAll,
  },
});

// Auto-generated selectors
const { selectAll, selectById, selectIds, selectEntities, selectTotal } =
  todosAdapter.getSelectors((state: RootState) => state.todos);

// Usage
const allTodos = useSelector(selectAll);              // Todo[]
const todo5 = useSelector(state => selectById(state, '5'));  // Todo | undefined
```

### Normalized State Shape

```text
Store: { todos: {
  ids: ['1', '2', '3'],
  entities: {
    '1': { id: '1', text: 'Buy milk', completed: false },
    '2': { id: '2', text: 'Read book', completed: true },
    '3': { id: '3', text: 'Exercise', completed: false },
  },
  loading: false,
}}
```

**Benefits of normalization:**
- O(1) lookup by ID — no array `.find()`
- Updates only touch one entity — no array iteration
- No duplicate data across features

---

## 11. Store Structure and Feature-Based Slices

### Recommended File Structure

```text
src/
  store/
    store.ts           ← configureStore
    hooks.ts           ← useAppDispatch, useAppSelector
  features/
    counter/
      counterSlice.ts
      Counter.tsx
      counterSelectors.ts
    posts/
      postsSlice.ts
      postsApi.ts      ← RTK Query
      PostsList.tsx
    auth/
      authSlice.ts
      authSelectors.ts
      Login.tsx
```

### Feature-Based Slice (Complete Example)

```ts
// features/auth/authSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

interface User { id: string; name: string; email: string; }
interface AuthState {
  user: User | null;
  token: string | null;
  status: 'idle' | 'loading' | 'succeeded' | 'failed';
  error: string | null;
}

export const login = createAsyncThunk(
  'auth/login',
  async (credentials: { email: string; password: string }, { rejectWithValue }) => {
    const res = await fetch('/api/login', {
      method: 'POST',
      body: JSON.stringify(credentials),
      headers: { 'Content-Type': 'application/json' },
    });
    if (!res.ok) return rejectWithValue('Invalid credentials');
    return await res.json() as { user: User; token: string };
  }
);

const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, token: null, status: 'idle', error: null } as AuthState,
  reducers: {
    logout(state) {
      state.user = null;
      state.token = null;
      state.status = 'idle';
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => { state.status = 'loading'; })
      .addCase(login.fulfilled, (state, action) => {
        state.status = 'succeeded';
        state.user = action.payload.user;
        state.token = action.payload.token;
      })
      .addCase(login.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload as string;
      });
  },
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;
```

---

## 12. DevTools Integration

RTK enables Redux DevTools Extension automatically in development. No configuration needed.

```ts
// DevTools enabled by default:
const store = configureStore({ reducer: { ... } });

// Explicitly control:
const store = configureStore({
  reducer: { ... },
  devTools: process.env.NODE_ENV !== 'production',
});
```

### DevTools Features

- **State snapshot** after every dispatched action
- **Action history** with type and payload
- **Time-travel debugging** — jump to any past state
- **State diff** — see exactly what changed
- **Import/Export** state for reproducing bugs
- **Dispatch** actions manually from the browser panel

### Named Actions for Better DevTools

With `createSlice`, action types are automatically named: `'sliceName/reducerName'`:

```ts
// Actions appear in DevTools as:
counter/increment
counter/incrementByAmount
users/fetchAll/pending
users/fetchAll/fulfilled
users/fetchAll/rejected
auth/login/fulfilled
```

---

## 13. RTK vs Context API vs Zustand

| Dimension | RTK | Context API | Zustand |
|---|---|---|---|
| Bundle size | ~16 kB (with RTK Query: ~30 kB) | 0 kB (built-in) | ~1.5 kB |
| Boilerplate | Moderate | Low (for simple state) | Minimal |
| Provider required | Yes (`<Provider>`) | Yes (`<Context.Provider>`) | No |
| DevTools | Excellent (Redux DevTools) | None built-in | Via middleware |
| Async handling | `createAsyncThunk`, RTK Query | Manual | Native async functions |
| Data fetching | RTK Query (powerful) | Manual | No built-in |
| Re-renders | Selector-based | All consumers | Selector-based |
| Immutability | Immer built-in | Manual | Manual (or Immer middleware) |
| Learning curve | Higher | Low | Low |
| TypeScript | Excellent | Good | Excellent |

### Decision Guide

**Use RTK when:**
- Large application with complex state across many features
- Need RTK Query for API caching (competitor to React Query)
- Team wants strict conventions and structured patterns
- Heavy DevTools time-travel debugging workflow
- Existing Redux codebase (migration path)

**Use Zustand when:**
- Small to medium applications
- Need minimal boilerplate and setup
- Want multiple independent stores per feature
- No RTK Query equivalent needed (using React Query instead)

**Use Context API when:**
- Simple, infrequently changing state (theme, locale, auth status)
- No external dependencies preferred
- Small number of consuming components

---

## 14. Redux Middleware

Middleware provides an extension point between dispatching an action and the moment it reaches the reducer.

### Middleware Signature

```ts
// A middleware is a curried function:
const myMiddleware = (storeAPI) => (next) => (action) => {
  // storeAPI.getState() — read current state
  // storeAPI.dispatch() — dispatch another action
  // next(action) — pass to next middleware / reducer
  // action — the dispatched action object

  console.log('Before:', action.type);
  const result = next(action);  // call the next middleware
  console.log('After:', storeAPI.getState());
  return result;
};
```

### Logging Middleware Example

```ts
const loggerMiddleware: Middleware = (storeAPI) => (next) => (action) => {
  if (process.env.NODE_ENV === 'development') {
    console.group(action.type);
    console.log('dispatching', action);
    const result = next(action);
    console.log('next state', storeAPI.getState());
    console.groupEnd();
    return result;
  }
  return next(action);
};

const store = configureStore({
  reducer: { ... },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(loggerMiddleware),
});
```

### Redux Thunk (Included by Default)

`redux-thunk` is included in RTK's default middleware. It allows dispatching functions (thunks) instead of plain action objects:

```ts
// Thunk: a function dispatched as an action
const loadUserData = (userId: string) => async (dispatch: AppDispatch, getState: () => RootState) => {
  const state = getState();
  if (state.users.entities[userId]) return;  // skip if already loaded

  dispatch(setLoading(true));
  const user = await fetchUser(userId);
  dispatch(addUser(user));
  dispatch(setLoading(false));
};

// Usage
dispatch(loadUserData('123'));
```

---

## 15. Common Mistakes

### Mutating State Outside of createSlice

```ts
// ❌ Wrong — mutating Redux state directly (outside Immer context)
const item = useSelector(state => state.cart.items[0]);
item.quantity = 5;  // Direct mutation — Redux won't detect this change!

// ✅ Correct — dispatch an action that updates via createSlice reducer
dispatch(updateItemQuantity({ id: item.id, quantity: 5 }));
```

### Over-Using Redux for Local UI State

```tsx
// ❌ Wrong — modal open state belongs in local component state, not Redux
const isModalOpen = useSelector(state => state.ui.isModalOpen);
dispatch(openModal());  // overkill for local UI

// ✅ Correct — local state for local UI concerns
const [isModalOpen, setIsModalOpen] = useState(false);
```

### Returning New Objects in useSelector

```tsx
// ❌ Wrong — new object reference every render → infinite re-render loop
const data = useSelector((state: RootState) => ({
  count: state.counter.value,
  step: state.counter.step,
}));

// ✅ Option A: Separate selectors
const count = useSelector((state: RootState) => state.counter.value);
const step = useSelector((state: RootState) => state.counter.step);

// ✅ Option B: shallowEqual
const data = useSelector(
  (state: RootState) => ({ count: state.counter.value, step: state.counter.step }),
  shallowEqual
);
```

### Forgetting to Add RTK Query Middleware

```ts
// ❌ Wrong — RTK Query cache won't work without the middleware
const store = configureStore({
  reducer: { [postsApi.reducerPath]: postsApi.reducer },
  // middleware not added — cache and polling won't work
});

// ✅ Correct
const store = configureStore({
  reducer: { [postsApi.reducerPath]: postsApi.reducer },
  middleware: (getDefault) => getDefault().concat(postsApi.middleware),
});
```

---

## 16. Best Practices

### Use TypedHooks Everywhere

Always use `useAppDispatch` and `useAppSelector` (typed wrappers) instead of raw `useDispatch` and `useSelector`. This avoids repetitive type annotations across every component.

### Co-locate Selectors with Slices

Define selector functions alongside the slice they access. Export them from the same file:

```ts
// counterSlice.ts — selectors exported from the same file
export const selectCount = (state: RootState) => state.counter.value;
export const selectStep = (state: RootState) => state.counter.step;
export const selectIsMax = createSelector(
  selectCount,
  (count) => count >= 100
);
```

### Use createSelector for Derived State

Avoid computing derived data in components. Use memoized selectors to prevent unnecessary re-renders.

### Normalize Complex State with createEntityAdapter

For any feature with a list of entities (users, posts, products), use `createEntityAdapter` for normalized O(1) lookup and efficient updates.

### Let RTK Query Replace createAsyncThunk for API Calls

For standard CRUD operations, RTK Query is far less code than `createAsyncThunk`:

```text
createAsyncThunk + extraReducers: ~40 lines per endpoint
RTK Query endpoint definition:   ~5 lines per endpoint
```

### Summary Reference

| Concern | Recommendation |
|---|---|
| Store setup | `configureStore` with feature reducers |
| Slice | `createSlice` — one per feature domain |
| TypeScript | Export `RootState`, `AppDispatch`; use `useAppSelector`, `useAppDispatch` |
| Mutations in reducers | Write direct mutations (Immer handles immutability) |
| Async | `createAsyncThunk` for custom async; RTK Query for API CRUD |
| Selectors | Co-locate with slice; use `createSelector` for derived state |
| useSelector objects | Use `shallowEqual` or separate selectors |
| Local UI state | Use `useState` — not Redux |
| API caching | RTK Query (replaces most `createAsyncThunk` usage) |
| Normalized data | `createEntityAdapter` for entity collections |
