## Redux Toolkit — Tricky Output Questions

> These questions test deep understanding of Redux Toolkit's createSlice, Immer integration, useSelector re-render behavior, createAsyncThunk lifecycle, RTK Query caching, and store subscription patterns. Each question reflects real interview and debugging scenarios.

---

## 1. createSlice

### Q1

```ts
import { createSlice } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment(state) {
      state.value += 1;
    },
    incrementByAmount(state, action) {
      state.value += action.payload;
    },
  },
});

console.log(counterSlice.actions.increment());
console.log(counterSlice.actions.incrementByAmount(5));
```

#### ❓ What is logged by each `console.log`?

<details>
<summary>✅ Answer</summary>

```txt
{ type: 'counter/increment', payload: undefined }
{ type: 'counter/incrementByAmount', payload: 5 }
```

**Explanation:** `createSlice` automatically generates action creators for each reducer function. The action type string is `'{sliceName}/{reducerName}'`. Calling `increment()` with no arguments generates an action with `payload: undefined`. Calling `incrementByAmount(5)` sets `payload: 5` — the argument is automatically assigned as `payload`. This is standard RTK behavior: the first argument to any action creator becomes `payload`.

</details>

---

### Q2

```ts
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0, step: 1 },
  reducers: {
    increment(state) {
      state.value += state.step;
      return state;  // ← returning state AND mutating
    },
  },
});
```

#### ❓ What happens when the `increment` reducer runs?

<details>
<summary>✅ Answer</summary>

```txt
Immer throws an error:
"An immer producer returned a new value *and* modified its draft.
Either return a new value *or* modify the draft."
```

**Explanation:** Immer enforces a strict rule: inside a producer function (the reducer), you must EITHER mutate the draft state OR return a new value — never both simultaneously. Mutating `state.value` uses the proxy mutation approach. Returning `state` (even though it's the same reference) tells Immer to use the return value as the new state. Combining both is ambiguous and Immer throws an error. Fix: either remove `return state` (mutation only) or don't mutate and return a new object.

</details>

---

### Q3

```ts
const todosSlice = createSlice({
  name: 'todos',
  initialState: [] as string[],
  reducers: {
    addTodo(state, action) {
      state.push(action.payload);
    },
    clearAll() {
      return [];  // ← returning new state instead of mutating
    },
  },
});

// Initial state
let state = todosSlice.reducer(undefined, { type: '@@INIT' });
state = todosSlice.reducer(state, todosSlice.actions.addTodo('Buy milk'));
state = todosSlice.reducer(state, todosSlice.actions.addTodo('Read book'));
state = todosSlice.reducer(state, todosSlice.actions.clearAll());

console.log(state);
```

#### ❓ What does `state` contain after all four reducer calls?

<details>
<summary>✅ Answer</summary>

```txt
[]
```

**Explanation:** The fourth call dispatches `clearAll()` which returns a new empty array `[]`. This replaces the entire state. In Immer, returning a value from a reducer replaces the state completely — the return value does not need to be a spread/merge. The final state is `[]`. Earlier calls added `'Buy milk'` and `'Read book'`, but `clearAll` wiped them.

</details>

---

### Q4

```ts
const usersSlice = createSlice({
  name: 'users',
  initialState: { list: [], count: 0 },
  reducers: {
    addUser(state, action) {
      state.list.push(action.payload);
      state.count = state.list.length;
    },
  },
});

let state = usersSlice.reducer(undefined, { type: '@@INIT' });
state = usersSlice.reducer(state, usersSlice.actions.addUser({ id: 1, name: 'Alice' }));
state = usersSlice.reducer(state, usersSlice.actions.addUser({ id: 2, name: 'Bob' }));

console.log(state.count);
console.log(state.list.length);
```

#### ❓ What is logged?

<details>
<summary>✅ Answer</summary>

```txt
2
2
```

**Explanation:** Both `addUser` calls run through Immer. Each call pushes to `state.list` and updates `state.count`. Immer applies mutations in order: after first call `count: 1, list.length: 1`; after second call `count: 2, list.length: 2`. Since we're inside Immer's proxy, both mutations are captured and applied to produce a new immutable state object. `count` correctly reflects `state.list.length` after each push.

</details>

---

### Q5

```ts
const initialState = { user: null, token: null };

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    reset() {
      return initialState;  // reference to the same initialState object
    },
    setUser(state, action) {
      state.user = action.payload;
    },
  },
});

let state = authSlice.reducer(undefined, { type: '@@INIT' });
state = authSlice.reducer(state, authSlice.actions.setUser({ id: 1, name: 'Alice' }));
state = authSlice.reducer(state, authSlice.actions.reset());

console.log(state === initialState);
console.log(state.user);
```

#### ❓ What is logged?

<details>
<summary>✅ Answer</summary>

```txt
false
null
```

**Explanation:** Returning `initialState` from a reducer returns a reference to the original object. However, Immer processes the return value and, because it returns a non-draft plain object, it performs a "structural share" — the resulting state is `initialState` (same reference). Actually: `state === initialState` is `true` in some Immer versions, but when Immer is involved with freezing, the returned object may be a different reference. The key point: `state.user` is `null` because `initialState.user` is `null`. Note: this pattern is safe as long as `initialState` is never mutated directly.

**Correction:** In practice with RTK's Immer: `state === initialState` may be `true` if Immer returns the same frozen object. The important answer is `state.user === null`.

</details>

---

## 2. useSelector

### Q6

```tsx
import { useSelector } from 'react-redux';

function Component() {
  const count = useSelector((state) => state.counter.value);
  const step = useSelector((state) => state.counter.step);

  console.log('render');
  return <div>{count}</div>;
}
```

The store updates: `counter.value` changes from `0` to `1`. `counter.step` stays `1`. How many times does "render" log?

#### ❓ How many re-renders occur?

<details>
<summary>✅ Answer</summary>

```txt
render  ← initial mount
render  ← after counter.value changes

Total: 2 renders
```

**Explanation:** `useSelector` uses strict reference equality (`===`) to compare the previous and new selector results. The first `useSelector` returns `state.counter.value` — this changes from `0` to `1`, so `0 === 1` is `false` → re-render triggered. The second `useSelector` returns `state.counter.step` — still `1`, so `1 === 1` is `true` → no additional re-render. When any `useSelector` in a component detects a change, the component re-renders once (not once per selector).

</details>

---

### Q7

```tsx
function ExpensiveComponent() {
  const filteredItems = useSelector((state) => ({
    items: state.cart.items.filter(i => i.active),
    count: state.cart.items.filter(i => i.active).length,
  }));

  console.log('render');
  return null;
}
```

An unrelated part of the store updates (e.g., `state.ui.theme` changes). Does `ExpensiveComponent` re-render?

#### ❓ Does the component re-render when an unrelated part of the store changes?

<details>
<summary>✅ Answer</summary>

```txt
Yes — ExpensiveComponent re-renders on EVERY store update.
```

**Explanation:** The selector returns a new object `{ items: [...], count: N }` on every invocation. `useSelector` compares the new return value against the previous using `===`. Since `{ items: [...], count: N } === { items: [...], count: N }` is always `false` (different object references), a re-render is triggered on every dispatch — even when `state.cart.items` hasn't changed. Fix with either: (a) two separate `useSelector` calls for `items` and `count`, (b) `shallowEqual` as the second argument, or (c) `createSelector` for memoization.

</details>

---

### Q8

```tsx
import { useSelector, shallowEqual } from 'react-redux';

function UserInfo() {
  const { name, email } = useSelector(
    (state) => ({ name: state.user.name, email: state.user.email }),
    shallowEqual
  );
  console.log('render');
  return <div>{name}</div>;
}
```

The store updates: `state.user.role` changes (name and email stay the same). Does `UserInfo` re-render?

#### ❓ Does shallowEqual prevent the re-render?

<details>
<summary>✅ Answer</summary>

```txt
No — UserInfo does NOT re-render.
```

**Explanation:** `shallowEqual` compares the keys of the returned object individually using `===`. The selector returns `{ name: state.user.name, email: state.user.email }`. Before and after the `role` update: `name` is the same string reference, `email` is the same string reference. `shallowEqual({ name: 'Alice', email: 'a@b.com' }, { name: 'Alice', email: 'a@b.com' })` returns `true`. Since the equality check passes, the component does NOT re-render. This is why `shallowEqual` is the fix for object selectors that would otherwise re-render on every dispatch.

</details>

---

### Q9

```ts
import { createSelector } from '@reduxjs/toolkit';

const selectItems = (state) => state.cart.items;
const selectActiveItems = createSelector(
  [selectItems],
  (items) => {
    console.log('selector ran');
    return items.filter(i => i.active);
  }
);
```

The store updates twice: first `state.ui.theme` changes, then `state.cart.items` changes. How many times does "selector ran" log?

#### ❓ How many times does the expensive filter computation run?

<details>
<summary>✅ Answer</summary>

```txt
selector ran  ← only when state.cart.items changes (2nd update)
Total: 1 time
```

**Explanation:** `createSelector` memoizes based on input selectors. `selectActiveItems` has one input selector: `selectItems`. The first update changes `state.ui.theme` — `selectItems(state)` returns the same `items` reference, so `createSelector` detects no change in inputs and returns the cached result WITHOUT running the filter function. The second update changes `state.cart.items` — `selectItems` returns a new array reference, so the memoization cache misses and the filter runs. This is the key benefit of `createSelector`.

</details>

---

### Q10

```tsx
function Counter() {
  const count = useSelector((state) => state.counter.value);
  const [localCount, setLocalCount] = useState(0);

  console.log('render');

  return (
    <div>
      <button onClick={() => setLocalCount(c => c + 1)}>Local +1</button>
      <p>{count} / {localCount}</p>
    </div>
  );
}
```

User clicks "Local +1" 3 times (Redux count stays at 0). How many re-renders?

#### ❓ How many times does "render" log after 3 local button clicks?

<details>
<summary>✅ Answer</summary>

```txt
render  ← initial mount
render  ← click 1
render  ← click 2
render  ← click 3

Total: 4 renders
```

**Explanation:** `useSelector` subscribes to Redux state changes. `useState` hooks cause re-renders on local state updates. These are completely independent. Clicking "Local +1" updates `localCount` via `setLocalCount`, triggering React re-renders through the normal `useState` mechanism. The `useSelector` subscription doesn't prevent re-renders caused by local state — it only avoids re-renders when the REDUX state hasn't changed. All 3 clicks cause re-renders because `localCount` changes each time.

</details>

---

## 3. createAsyncThunk

### Q11

```ts
const fetchUser = createAsyncThunk(
  'users/fetchById',
  async (userId: string) => {
    const res = await fetch(`/api/users/${userId}`);
    return await res.json();
  }
);

const usersSlice = createSlice({
  name: 'users',
  initialState: { user: null, status: 'idle' },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchUser.pending, (state) => {
        state.status = 'loading';
        console.log('pending');
      })
      .addCase(fetchUser.fulfilled, (state, action) => {
        state.user = action.payload;
        state.status = 'success';
        console.log('fulfilled');
      });
  },
});

store.dispatch(fetchUser('1'));
```

The API responds successfully. What is logged and in what order?

#### ❓ What is the order of pending and fulfilled logs?

<details>
<summary>✅ Answer</summary>

```txt
pending    ← dispatched synchronously when fetchUser('1') is called
fulfilled  ← dispatched asynchronously when the API responds
```

**Explanation:** `createAsyncThunk` dispatches three action types in sequence. When `dispatch(fetchUser('1'))` is called: (1) `fetchUser.pending` is dispatched immediately and synchronously — the `'loading'` status is set and "pending" logs. (2) The `async` function runs (network request). (3) After the Promise resolves, `fetchUser.fulfilled` is dispatched and "fulfilled" logs. `fetchUser.rejected` would fire if the Promise rejects (not in this example).

</details>

---

### Q12

```ts
const fetchPosts = createAsyncThunk(
  'posts/fetchAll',
  async (_, { rejectWithValue }) => {
    const res = await fetch('/api/posts');
    if (!res.ok) {
      return rejectWithValue({ status: res.status, message: 'Failed' });
    }
    return await res.json();
  }
);
```

The API returns a 404 response. Which lifecycle action is dispatched and what is `action.payload`?

#### ❓ What action fires and what does its payload contain?

<details>
<summary>✅ Answer</summary>

```txt
'posts/fetchAll/rejected' is dispatched.
action.payload = { status: 404, message: 'Failed' }
action.error = { message: 'Rejected' }  (standard error from RTK)
```

**Explanation:** `rejectWithValue(value)` causes the thunk to dispatch the `rejected` action with `action.payload` set to the provided value. Without `rejectWithValue`, if you `throw new Error('...')`, `action.error.message` would contain the error message but `action.payload` would be `undefined`. `rejectWithValue` is the idiomatic RTK way to pass structured error data from a thunk to your reducer for display in the UI.

</details>

---

### Q13

```ts
const fetchData = createAsyncThunk(
  'data/fetch',
  async () => {
    await new Promise(r => setTimeout(r, 100));
    return { value: 42 };
  }
);

// Dispatch twice quickly
const promise1 = store.dispatch(fetchData());
const promise2 = store.dispatch(fetchData());
```

#### ❓ How many times does the async function run and how many times do `pending` and `fulfilled` actions dispatch?

<details>
<summary>✅ Answer</summary>

```txt
Async function runs: 2 times
pending dispatches: 2 times
fulfilled dispatches: 2 times

Total dispatches: 4 (2 pending + 2 fulfilled)
```

**Explanation:** Each `dispatch(fetchData())` call creates an independent thunk invocation. There is no automatic deduplication — both run concurrently. Both dispatch their own `pending` action immediately, then both dispatch their own `fulfilled` action after ~100ms. The second `fulfilled` dispatches after the first. To prevent duplicate requests, use `condition` in the thunk options or check state in the thunk before making the request.

</details>

---

### Q14

```ts
const updateUser = createAsyncThunk(
  'users/update',
  async (data: { id: string; name: string }, { getState }) => {
    const state = getState() as RootState;
    const currentUser = state.auth.user;
    
    if (!currentUser) {
      throw new Error('Not authenticated');
    }
    
    return await updateUserApi(data);
  }
);
```

The user is not authenticated (`state.auth.user = null`). `dispatch(updateUser({ id: '1', name: 'Alice' }))` is called. What happens?

#### ❓ Which lifecycle action fires when the thunk throws?

<details>
<summary>✅ Answer</summary>

```txt
'users/update/pending' fires first (synchronously).
Then 'users/update/rejected' fires.
action.error.message = "Not authenticated"
action.payload = undefined (throw was used, not rejectWithValue)
```

**Explanation:** When the async function `throws` (rather than using `rejectWithValue`), `createAsyncThunk` catches the error and dispatches the `rejected` action with `action.error` set to the serialized error object. `action.error.message` contains `'Not authenticated'`. `action.payload` is `undefined` when using `throw` — use `rejectWithValue` instead to put data in `action.payload`. In the reducer's `addCase(updateUser.rejected, ...)`, check `action.error.message` (not `action.payload`) in this case.

</details>

---

### Q15

```ts
const fetchPosts = createAsyncThunk('posts/fetch', async () => {
  const res = await fetch('/api/posts');
  return await res.json();
});

const postsSlice = createSlice({
  name: 'posts',
  initialState: { data: [], status: 'idle' },
  reducers: {},
  extraReducers: (builder) => {
    builder
      .addCase(fetchPosts.fulfilled, (state, action) => {
        state.data = action.payload;
        state.status = 'success';
      })
      .addMatcher(
        (action) => action.type.endsWith('/pending'),
        (state) => { state.status = 'loading'; }
      )
      .addDefaultCase((state) => {
        console.log('default case');
      });
  },
});
```

`fetchPosts.pending` is dispatched. Which cases run?

#### ❓ Does `addCase`, `addMatcher`, or `addDefaultCase` handle `fetchPosts.pending`?

<details>
<summary>✅ Answer</summary>

```txt
addMatcher runs — because the action type 'posts/fetch/pending' ends with '/pending'.
addDefaultCase does NOT run for matched actions.
```

**Explanation:** RTK's `extraReducers` builder processes cases in order: `addCase` (exact match) → `addMatcher` (predicate match) → `addDefaultCase` (fallback). `fetchPosts.pending` has type `'posts/fetch/pending'`. No `addCase` handles this exact type. The `addMatcher` predicate checks `action.type.endsWith('/pending')` → `true` → this case runs, setting `status: 'loading'`. `addDefaultCase` only runs when NO other case or matcher matches, so it's skipped here. Note: multiple `addMatcher` cases can match the same action and all will run.

</details>

---

## 4. RTK Query

### Q16

```ts
export const postsApi = createApi({
  reducerPath: 'postsApi',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['Post'],
  endpoints: (builder) => ({
    getPosts: builder.query({
      query: () => '/posts',
      providesTags: ['Post'],
    }),
    createPost: builder.mutation({
      query: (post) => ({ url: '/posts', method: 'POST', body: post }),
      invalidatesTags: ['Post'],
    }),
  }),
});
```

```tsx
function App() {
  const { data: posts } = useGetPostsQuery();
  const [createPost] = useCreatePostMutation();

  // User creates a post
  createPost({ title: 'New' });
}
```

#### ❓ What happens to the `useGetPostsQuery` cache after `createPost` resolves?

<details>
<summary>✅ Answer</summary>

```txt
The ['Post'] tag is invalidated.
RTK Query automatically refetches the getPosts query.
The posts list updates with the new post included.
```

**Explanation:** `invalidatesTags: ['Post']` in `createPost` tells RTK Query that after a successful mutation, all queries that `providesTags: ['Post']` are now stale and should be refetched. Since `getPosts` provides the `'Post'` tag, it is automatically re-fetched. The component receives updated data without any manual cache management. This tag-based invalidation is RTK Query's automated alternative to React Query's `invalidateQueries`.

</details>

---

### Q17

```tsx
function PostDetail({ id }) {
  const { data } = useGetPostQuery(id);  // id changes from 1 to 2
  return <div>{data?.title}</div>;
}
```

The `id` prop changes from `1` to `2`. What happens to the cached data for post `1`?

#### ❓ Does the cache entry for `id: 1` stay in the RTK Query cache?

<details>
<summary>✅ Answer</summary>

```txt
Yes — the cache entry for id: 1 stays in the cache (for a period).
A new request is made for id: 2.
The cache for id: 1 is kept for keepUnusedDataFor seconds (default: 60 seconds).
```

**Explanation:** When `id` changes from `1` to `2`, the query key changes from `['getPost', 1]` to `['getPost', 2]`. RTK Query makes a new request for id `2`. The cache entry for id `1` is not immediately deleted — it's kept for `keepUnusedDataFor` seconds (default 60) in case the component subscribes back to it (e.g., user navigates back). This prevents flickering when navigating between posts. After the TTL expires with no subscribers, it's garbage collected.

</details>

---

### Q18

```ts
getPosts: builder.query({
  query: () => '/posts',
  providesTags: (result) =>
    result
      ? [
          ...result.map(({ id }) => ({ type: 'Post' as const, id })),
          { type: 'Post', id: 'LIST' },
        ]
      : [{ type: 'Post', id: 'LIST' }],
}),

deletePost: builder.mutation({
  query: (id) => ({ url: `/posts/${id}`, method: 'DELETE' }),
  invalidatesTags: (result, error, id) => [{ type: 'Post', id }],
}),
```

User deletes post with id `3`. Which queries are refetched?

#### ❓ Does `deletePost(3)` cause `getPosts` to refetch?

<details>
<summary>✅ Answer</summary>

```txt
Yes — getPosts is refetched.

The deletePost mutation invalidates { type: 'Post', id: 3 }.
getPosts provides [{ type: 'Post', id: 1 }, { type: 'Post', id: 3 }, ..., { type: 'Post', id: 'LIST' }].
Since { type: 'Post', id: 3 } is in getPosts' provided tags, it is invalidated.
```

**Explanation:** RTK Query's tag matching: when `deletePost(3)` invalidates `{ type: 'Post', id: 3 }`, any cached query that **provides** a tag matching `{ type: 'Post', id: 3 }` is invalidated. Since `getPosts` provides individual `{ type: 'Post', id }` tags for each post (including id: 3), it gets invalidated and refetches. The `{ type: 'Post', id: 'LIST' }` tag is used to also invalidate getPosts when a new post is created (since creating adds an item but doesn't have a specific ID to match).

</details>

---

### Q19

```tsx
function PostsList() {
  const { data, isLoading } = useGetPostsQuery(undefined, {
    pollingInterval: 5000,
  });
  return <div>{data?.length} posts</div>;
}

function App() {
  const [show, setShow] = useState(true);
  return (
    <>
      {show && <PostsList />}
      <button onClick={() => setShow(false)}>Hide</button>
    </>
  );
}
```

User clicks "Hide". What happens to the polling?

#### ❓ Does polling continue after `PostsList` unmounts?

<details>
<summary>✅ Answer</summary>

```txt
No — polling stops when PostsList unmounts.
The subscription count drops to zero and the interval is cleared.
```

**Explanation:** RTK Query's `pollingInterval` is tied to active subscriptions. When `PostsList` unmounts, the component unsubscribes from the `getPosts` query. With zero active subscribers, the polling interval is cancelled. The cached data remains (for `keepUnusedDataFor` seconds) but is no longer actively updated. When `PostsList` mounts again, polling restarts from the point of re-subscription.

</details>

---

### Q20

```ts
updatePost: builder.mutation({
  query: ({ id, changes }) => ({
    url: `/posts/${id}`,
    method: 'PATCH',
    body: changes,
  }),
  async onQueryStarted({ id, changes }, { dispatch, queryFulfilled }) {
    // Optimistic update
    const patchResult = dispatch(
      postsApi.util.updateQueryData('getPosts', undefined, (draft) => {
        const post = draft.find(p => p.id === id);
        if (post) Object.assign(post, changes);
      })
    );
    try {
      await queryFulfilled;
    } catch {
      patchResult.undo();  // rollback on error
    }
  },
}),
```

The PATCH API call fails. What does `patchResult.undo()` do?

#### ❓ What happens to the optimistic update when `patchResult.undo()` is called?

<details>
<summary>✅ Answer</summary>

```txt
patchResult.undo() reverts the getPosts cache to its state before the optimistic update.
The UI immediately shows the original post data (before the changes).
```

**Explanation:** `postsApi.util.updateQueryData` applies an optimistic update to the RTK Query cache using Immer — it immediately updates the `getPosts` cache entry with the changed values. The `patchResult` object contains an `undo()` method that dispatches a reverse patch to restore the previous cache state. When the API call fails (caught in the `try/catch`), `patchResult.undo()` is called to roll back the optimistic change. This is RTK Query's built-in optimistic update pattern, equivalent to React Query's `onMutate` + rollback pattern.

</details>

---

## 5. Edge Cases

### Q21

```ts
const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment(state) {
      state.value += 1;
    },
  },
});

const store = configureStore({
  reducer: { counter: counterSlice.reducer },
});

// Outside React — direct store interaction
const stateRef = store.getState().counter;
store.dispatch(counterSlice.actions.increment());
console.log(stateRef === store.getState().counter);
console.log(stateRef.value);
console.log(store.getState().counter.value);
```

#### ❓ What is logged for all three console statements?

<details>
<summary>✅ Answer</summary>

```txt
false         ← different object references (Immer produced a new object)
0             ← stateRef captured the old state before dispatch
1             ← new state after dispatch
```

**Explanation:** Immer always produces a new object when state is mutated inside a reducer. After `dispatch(increment())`, `store.getState().counter` returns a new object (different reference from `stateRef`). `stateRef` still holds a reference to the old state snapshot `{ value: 0 }`. `store.getState().counter` returns the new state `{ value: 1 }`. This immutability guarantee is fundamental to Redux — state snapshots are stable references you can compare and hold.

</details>

---

### Q22

```ts
const rootReducer = combineReducers({
  counter: counterReducer,
  user: userReducer,
});

// Dispatch an action that only counter handles
store.dispatch(counterSlice.actions.increment());
```

#### ❓ Which reducers run when `increment()` is dispatched?

<details>
<summary>✅ Answer</summary>

```txt
BOTH counterReducer and userReducer run.
All reducers run on every dispatch.
```

**Explanation:** This is a fundamental Redux principle: on every dispatch, Redux passes the action to ALL reducers. `combineReducers` calls each sub-reducer with its slice of state. `counterReducer` handles the `'counter/increment'` action and updates state. `userReducer` receives the same action — since it doesn't have a case for `'counter/increment'`, it falls through to the default case and returns the current state unchanged. Every reducer receives every action; reducers simply ignore actions they don't care about.

</details>

---

### Q23

```tsx
function CounterA() {
  const value = useSelector((state) => state.counter.value);
  console.log('CounterA render');
  return <div>{value}</div>;
}

function CounterB() {
  const doubled = useSelector((state) => state.counter.value * 2);
  console.log('CounterB render');
  return <div>{doubled}</div>;
}

// store.dispatch(counterSlice.actions.increment());
// counter.value changes from 0 to 1
```

#### ❓ After the dispatch, which components re-render and what do they display?

<details>
<summary>✅ Answer</summary>

```txt
CounterA render  ← 0 → 1 (selector result changed)
CounterB render  ← 0 → 2 (selector result changed: 0*2=0 vs 1*2=2)

Both components re-render.
CounterA displays: 1
CounterB displays: 2
```

**Explanation:** Both components subscribed via `useSelector`. When `counter.value` changes from `0` to `1`: CounterA's selector returns `1` (was `0`) → `0 === 1` is `false` → re-renders showing `1`. CounterB's selector returns `2` (was `0`) → `0 === 2` is `false` → re-renders showing `2`. Both selectors detected changes and both components re-render. If `counter.value` were `0 → 0` (no change), neither would re-render.

</details>

---

### Q24

```tsx
function App() {
  const dispatch = useDispatch();
  const items = useSelector((state) => state.cart.items);

  function handleBadUpdate() {
    // Trying to mutate outside of a reducer
    items.push({ id: 999, name: 'Hacked item' });
    console.log('items length:', items.length);
  }

  return <button onClick={handleBadUpdate}>Bad Update</button>;
}
```

User clicks the button. What happens?

#### ❓ Does the mutation succeed? Does the component re-render? What is logged?

<details>
<summary>✅ Answer</summary>

```txt
Error thrown: "Cannot add property ..., object is not extensible"
(or similar TypeError — Redux DevTools freezes state in development)
```

**Explanation:** RTK's `configureStore` enables `serializabilityCheck` middleware and, in development, freezes state using `Object.freeze` via Immer. The `items` array returned by `useSelector` is a frozen object. Calling `.push()` on a frozen array throws a `TypeError` in development. In production (without freezing), the push would modify the array in memory BUT Redux's subscription system would NOT detect it (no dispatch occurred), so the component would NOT re-render and the store state would be inconsistent. Always dispatch actions to update Redux state — never mutate directly.

</details>

---

### Q25

```ts
const store = configureStore({
  reducer: { counter: counterReducer },
});

// Subscribe to store changes outside React
const unsubscribe = store.subscribe(() => {
  const state = store.getState();
  console.log('New count:', state.counter.value);
});

store.dispatch(counterSlice.actions.increment());  // count: 1
store.dispatch(counterSlice.actions.increment());  // count: 2
unsubscribe();
store.dispatch(counterSlice.actions.increment());  // count: 3

console.log('Final:', store.getState().counter.value);
```

#### ❓ What is logged?

<details>
<summary>✅ Answer</summary>

```txt
New count: 1
New count: 2
Final: 3
```

**Explanation:** `store.subscribe(callback)` registers a listener that fires after every dispatch. The first two dispatches trigger the subscription: "New count: 1" and "New count: 2". `unsubscribe()` removes the listener. The third dispatch increments `counter.value` to `3` but the subscription no longer fires — nothing is logged for it. `store.getState().counter.value` is `3` because the state WAS updated (dispatch still works after unsubscribe). The subscription was removed but the reducer still processes actions normally.

</details>

---

## Topics Covered

| Category | Questions | Concepts Tested |
|---|---|---|
| createSlice | Q1–Q5 | Auto-generated action types, mutate-OR-return rule, clearAll with return, Immer nested mutations, initialState reference |
| useSelector | Q6–Q10 | Re-render on specific change, new object reference problem, shallowEqual, createSelector memoization, local vs Redux state |
| createAsyncThunk | Q11–Q15 | Lifecycle action order, rejectWithValue vs throw, duplicate dispatch, getState in thunk, addMatcher vs addCase |
| RTK Query | Q16–Q20 | Tag invalidation and refetch, cache on arg change, granular tag matching, polling on unmount, optimistic update rollback |
| Edge Cases | Q21–Q25 | Immer new object reference, all reducers run on dispatch, multiple selector re-renders, frozen state mutation error, store.subscribe |
