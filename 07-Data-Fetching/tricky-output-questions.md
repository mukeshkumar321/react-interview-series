## Data Fetching — Tricky Output Questions

> These questions focus on race conditions, stale closures in effects, AbortController, React Query behavior, caching, deduplication, and edge cases in data fetching patterns. Each question reflects a real scenario you may encounter in a senior React interview.

---

## 1. useEffect Fetching

### Q1

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

#### ❓ userId changes from 1 to 2 rapidly. What is the potential bug?

<details>
<summary>✅ Answer</summary>

```txt
Race condition: the response for userId=1 may arrive AFTER the response
for userId=2, overwriting the correct data with stale data.
```

**Explanation:** Both fetches are in flight simultaneously. If the network response for `userId=1` arrives last, `setUser` is called with Alice's data even though the component is now supposed to show userId=2's profile. The user sees wrong data with no error. The fix is to use an `ignore` flag in cleanup or `AbortController`.

</details>

---

### Q2

```jsx
function Profile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    let ignore = false;

    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        if (!ignore) setUser(data);
      });

    return () => { ignore = true; };
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

#### ❓ userId changes from 1 to 2. Does the fetch for userId=1 get cancelled? What does `ignore` actually prevent?

<details>
<summary>✅ Answer</summary>

```txt
The network request for userId=1 is NOT cancelled — it still completes.
The `ignore` flag only prevents the stale response from calling setUser.
```

**Explanation:** When `userId` changes to 2, the cleanup of the first effect sets `ignore = true`. If the first fetch resolves late, `if (!ignore)` is false so `setUser` is never called — preventing stale data from rendering. However, the HTTP request travels to the server and back. Only `AbortController` can cancel the network request itself and save bandwidth.

</details>

---

### Q3

```jsx
function PostList({ categoryId }) {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch(`/api/posts?cat=${categoryId}`)
      .then(res => res.json())
      .then(data => setPosts(data));
  }, []); // empty deps
}
```

#### ❓ The user selects a different category. What happens?

<details>
<summary>✅ Answer</summary>

```txt
The fetch does NOT re-run. The component displays posts from the
original categoryId forever, regardless of prop changes.
```

**Explanation:** The empty dependency array `[]` means "run once on mount." The closure captures `categoryId` at mount time. When the parent passes a new `categoryId`, the component re-renders with the new prop value, but the effect never fires again. The ESLint `exhaustive-deps` rule flags this as a missing dependency. Fix: `}, [categoryId])`.

</details>

---

### Q4

```jsx
useEffect(async () => {
  const res = await fetch('/api/data');
  const json = await res.json();
  setData(json);
}, []);
```

#### ❓ What warning does React emit and why?

<details>
<summary>✅ Answer</summary>

```txt
Warning: An effect function must not return anything besides a function,
which is used for clean-up. You returned: [object Promise]
```

**Explanation:** `async` functions always return a Promise. `useEffect` expects the callback to return either `undefined` or a cleanup function. A Promise is neither. React cannot run it as cleanup, so it ignores it and logs a warning. The deeper problem is that no cleanup can be registered — if the component unmounts before the fetch completes, `setData` is called on an unmounted component. Fix: define an inner async function and call it immediately.

```jsx
useEffect(() => {
  async function load() {
    const res = await fetch('/api/data');
    const json = await res.json();
    setData(json);
  }
  load();
}, []);
```

</details>

---

### Q5

```jsx
function Search({ query }) {
  const [results, setResults] = useState([]);

  useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(res => res.json())
      .then(data => setResults(data))
      .catch(err => {
        if (err.name === 'AbortError') return;
        console.error(err);
      });

    return () => controller.abort();
  }, [query]);

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}
```

#### ❓ What happens when `query` changes from "react" to "redux" before the first fetch completes?

<details>
<summary>✅ Answer</summary>

```txt
The cleanup for the "react" effect fires, calling controller.abort().
The in-flight fetch for "react" is aborted at the network level.
An AbortError is thrown, caught by the .catch handler, and silently ignored.
The new effect fires for "redux" with a fresh AbortController.
```

**Explanation:** `AbortController` cancels the actual HTTP request — not just the state update. The `.catch` guard checks `err.name === 'AbortError'` to distinguish intentional aborts from real errors. This is the cleanest solution for preventing race conditions in data fetching.

</details>

---

## 2. Loading and Error States

### Q6

```jsx
function Dashboard() {
  const [users, setUsers] = useState([]);
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    setLoading(true);

    fetch('/api/users')
      .then(r => r.json())
      .then(data => {
        setUsers(data);
        setLoading(false);
      });

    fetch('/api/posts')
      .then(r => r.json())
      .then(data => {
        setPosts(data);
        setLoading(false);
      });
  }, []);
}
```

#### ❓ Both fetches are in flight. The users fetch finishes first. What is displayed? What is the bug?

<details>
<summary>✅ Answer</summary>

```txt
When users fetch completes: setLoading(false) is called.
The spinner disappears even though posts are still loading.
When posts complete: setLoading(false) is called again (no-op).

Bug: A single loading flag cannot represent two independent async operations.
```

**Explanation:** `loading` becomes `false` as soon as the fastest request completes, misleading the user into thinking all data is ready. The posts data renders as an empty list briefly. Fix: use separate loading flags per resource, or use `Promise.all` to wait for both, or use a counter.

```jsx
const [loadingCount, setLoadingCount] = useState(0);
// increment before each fetch, decrement in each .then
// loading = loadingCount > 0
```

</details>

---

### Q7

```jsx
function UserCard({ userId }) {
  const [user, setUser] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, [userId]);

  if (error) return <p>Error: {error}</p>;
  return <p>{user?.name}</p>;
}
```

#### ❓ The server returns a 404 with `{ message: "Not Found" }`. What does the user see?

<details>
<summary>✅ Answer</summary>

```txt
The user sees nothing — the error state is never set.
The component renders with user = null indefinitely, showing an empty paragraph.
```

**Explanation:** `fetch` does NOT reject on HTTP error status codes (4xx, 5xx). It only rejects on network failures. `res.json()` successfully parses the `{ message: "Not Found" }` body, and `setUser({ message: "Not Found" })` is called. `user?.name` is `undefined`. Fix: check `res.ok` before parsing.

```jsx
.then(res => {
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
})
.then(data => setUser(data))
.catch(err => setError(err.message));
```

</details>

---

### Q8

```jsx
class App extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) return <h1>Something went wrong</h1>;
    return this.props.children;
  }
}

function DataWidget() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch('/api/data')
      .then(res => res.json())
      .then(setData)
      .catch(err => { throw err; }); // rethrow
  }, []);

  return <div>{data?.value}</div>;
}
```

#### ❓ The fetch fails with a network error. Does the Error Boundary catch it?

<details>
<summary>✅ Answer</summary>

```txt
No. The Error Boundary does NOT catch the error.
```

**Explanation:** Error Boundaries only catch errors that occur during rendering, in lifecycle methods, and in constructors. Errors thrown inside asynchronous callbacks (like `.catch(() => { throw err })`) are not caught by Error Boundaries because they run outside React's rendering cycle. The unhandled rejection surfaces as an uncaught promise rejection. To propagate an async error to an Error Boundary, you must use a pattern that throws during render:

```jsx
const [error, setError] = useState(null);
if (error) throw error; // throws during render → caught by Error Boundary
.catch(err => setError(err)); // store it in state, let render throw
```

</details>

---

### Q9

```jsx
function Page() {
  const [userLoading, setUserLoading] = useState(true);
  const [postsLoading, setPostsLoading] = useState(true);

  useEffect(() => {
    fetchUser().finally(() => setUserLoading(false));
  }, []);

  useEffect(() => {
    fetchPosts().finally(() => setPostsLoading(false));
  }, []);

  const isLoading = userLoading || postsLoading;

  return isLoading ? <Spinner /> : <Content />;
}
```

#### ❓ Is there a flash of `<Content />` before both requests finish?

<details>
<summary>✅ Answer</summary>

```txt
No. isLoading is true until BOTH flags are false.
The spinner shows until the slower of the two requests completes.
```

**Explanation:** `isLoading = userLoading || postsLoading`. Both start as `true`. After userFetch completes, `userLoading = false` but `postsLoading` is still `true`, so `isLoading` remains `true`. Content renders only when both are `false`. This is the correct pattern for parallel fetches with a unified loading gate.

</details>

---

### Q10

```jsx
function ProductPage({ id }) {
  const [product, setProduct] = useState(null);
  const [reviews, setReviews] = useState(null);

  useEffect(() => {
    fetchProduct(id).then(setProduct);
  }, [id]);

  useEffect(() => {
    if (!product) return;
    fetchReviews(product.reviewListId).then(setReviews);
  }, [product]);

  return <div>{product?.name} — {reviews?.length} reviews</div>;
}
```

#### ❓ Describe the order of requests and re-renders. Is this a waterfall?

<details>
<summary>✅ Answer</summary>

```txt
Yes, this is a waterfall:

1. Mount → fetchProduct fires
2. product state updates → re-render → second useEffect runs → fetchReviews fires
3. reviews state updates → re-render → component renders with both

Total time = fetchProduct time + fetchReviews time (sequential)
```

**Explanation:** `fetchReviews` depends on `product.reviewListId`, so it cannot start until `fetchProduct` resolves. The two requests cannot run in parallel. Fix options: (1) change the API to return product and reviews in one request, (2) pass `reviewListId` as a direct prop alongside `id` to enable parallel fetching, or (3) use React Query's `enabled` option with a parallel query that has the ID upfront.

</details>

---

## 3. React Query / TanStack Query Behavior

### Q11

```jsx
const { data } = useQuery({
  queryKey: ['user', 1],
  queryFn: () => fetchUser(1),
  staleTime: 5 * 60 * 1000, // 5 minutes
});
```

#### ❓ The user navigates away and returns 3 minutes later. What happens?

<details>
<summary>✅ Answer</summary>

```txt
The cached data is returned immediately (no loading state).
No refetch occurs because the data is within the 5-minute staleTime window.
The user sees the cached data instantly.
```

**Explanation:** `staleTime` defines how long data is considered "fresh." Within the stale window, React Query serves cached data without refetching, even if the component remounts. After 5 minutes, the data becomes stale — the next access triggers a background refetch while showing the stale data immediately (stale-while-revalidate). Default `staleTime` is `0`, meaning all data is immediately stale.

</details>

---

### Q12

```jsx
// Component A
const { data: user } = useQuery({
  queryKey: ['user', 1],
  queryFn: () => fetchUser(1),
});

// Component B (mounts 2 seconds later on the same page)
const { data: user } = useQuery({
  queryKey: ['user', 1],
  queryFn: () => fetchUser(1),
});
```

#### ❓ How many HTTP requests are made to `/api/user/1`?

<details>
<summary>✅ Answer</summary>

```txt
One request total (assuming default staleTime: 0 and the data is still
in the cache when Component B mounts).
```

**Explanation:** React Query deduplicates queries by `queryKey`. When Component A mounts, a request fires. When Component B mounts 2 seconds later, React Query sees that `['user', 1]` already has cached data. With `staleTime: 0` (default), the data is stale, so React Query triggers a background refetch — but only one, shared refetch. If both mounted simultaneously, only one network request would fire at all (request deduplication).

</details>

---

### Q13

```jsx
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  refetchOnWindowFocus: true, // default
});
```

#### ❓ A user has the tab open for 10 minutes, switches to another tab to read an article, then returns. What happens?

<details>
<summary>✅ Answer</summary>

```txt
React Query detects the window focus event and triggers a background refetch.
The stale cached data is displayed immediately.
When the refetch completes, the UI updates with fresh data.
The user never sees a loading spinner.
```

**Explanation:** `refetchOnWindowFocus` is `true` by default. When the browser window/tab regains focus, React Query refetches all stale queries. This keeps data fresh for long-lived sessions without requiring manual invalidation. The user sees the old data instantly, then the UI silently updates — this is the stale-while-revalidate pattern.

</details>

---

### Q14

```jsx
const { mutate: updateTodo } = useMutation({
  mutationFn: (todo) => api.updateTodo(todo),
  onMutate: async (updatedTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    const previousTodos = queryClient.getQueryData(['todos']);

    queryClient.setQueryData(['todos'], (old) =>
      old.map(t => t.id === updatedTodo.id ? updatedTodo : t)
    );

    return { previousTodos };
  },
  onError: (err, updatedTodo, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

#### ❓ The mutation fails with a 500 error. What does the user see? Walk through the execution order.

<details>
<summary>✅ Answer</summary>

```txt
1. mutate() called
2. onMutate fires: cancel in-flight todos queries, save snapshot,
   optimistically apply the update → UI shows updated todo immediately
3. Server returns 500
4. onError fires: roll back to previousTodos → UI reverts to original state
5. onSettled fires: invalidate ['todos'] → background refetch confirms server state
```

**Explanation:** This is the full optimistic update pattern with rollback. `onMutate` saves a snapshot via `getQueryData` before applying the optimistic update. If the mutation fails, `onError` restores the snapshot via `setQueryData`. `onSettled` always fires (success or error) and triggers a refetch to ensure the UI matches server truth. The user sees an instant UI update, then a revert if the server rejects it.

</details>

---

### Q15

```jsx
const { data, isLoading, isFetching } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
});
```

#### ❓ What is the difference between `isLoading` and `isFetching`? When is each true?

<details>
<summary>✅ Answer</summary>

```txt
isLoading: true when there is NO cached data AND a fetch is in progress.
           This is the "hard loading" state — no data to show yet.

isFetching: true whenever a fetch is in progress, regardless of
            whether cached data exists.
            This includes background refetches.
```

**Explanation:** On first load (no cache): both `isLoading` and `isFetching` are `true`. On a background refetch (stale data, window focus): `isLoading` is `false` (cached data exists), `isFetching` is `true`. This distinction lets you show a skeleton on first load (`isLoading`) vs. a subtle spinner on background updates (`isFetching`). In TanStack Query v5, `isLoading` is replaced by `isPending` for mutations and has been renamed — but the same concept applies.

</details>

---

## 4. Caching and Deduplication

### Q16

```jsx
// In ComponentA.jsx
function ComponentA() {
  const { data } = useQuery({
    queryKey: ['config'],
    queryFn: fetchConfig,
  });
  return <div>{data?.theme}</div>;
}

// In ComponentB.jsx
function ComponentB() {
  const { data } = useQuery({
    queryKey: ['config'],
    queryFn: fetchConfig,
  });
  return <div>{data?.language}</div>;
}

// In App.jsx
function App() {
  return (
    <>
      <ComponentA />
      <ComponentB />
    </>
  );
}
```

#### ❓ How many times does `fetchConfig` run when App first mounts?

<details>
<summary>✅ Answer</summary>

```txt
Once. React Query deduplicates concurrent queries with the same queryKey.
```

**Explanation:** Both components use `queryKey: ['config']`. When they mount simultaneously, React Query sees that both want the same data and fires a single network request. Both components subscribe to the same cache entry and both receive the data when it resolves. This is automatic request deduplication — a key benefit over rolling your own `useEffect` fetch.

</details>

---

### Q17

```jsx
function useUserData(userId) {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    gcTime: 0, // was cacheTime in v4
  });
}
```

#### ❓ A component using this hook unmounts. 1 second later, a different component mounts and calls `useUserData(1)`. What happens?

<details>
<summary>✅ Answer</summary>

```txt
A fresh network request is made. The cache entry was garbage-collected
immediately when the last subscriber unmounted (gcTime: 0).
```

**Explanation:** `gcTime` (formerly `cacheTime`) controls how long inactive cache entries persist after all subscribers unmount. With `gcTime: 0`, the cache entry is destroyed immediately on unmount. The new component finds no cache and triggers a fresh fetch. The default `gcTime` is 5 minutes, during which unmounted data remains in cache for instant access on remount.

</details>

---

### Q18

```jsx
async function addTodo(newTodo) {
  await api.createTodo(newTodo);
  queryClient.invalidateQueries({ queryKey: ['todos'] });
}

async function deleteTodo(id) {
  await api.deleteTodo(id);
  queryClient.invalidateQueries({ queryKey: ['todos'] });
}
```

#### ❓ After `addTodo` succeeds, what exactly happens to the `['todos']` cache entry?

<details>
<summary>✅ Answer</summary>

```txt
1. The cache entry for ['todos'] is marked as stale.
2. If any component is currently subscribed to ['todos'], a background
   refetch is triggered immediately.
3. The UI updates when the refetch resolves with fresh data from the server.
4. If no component is subscribed, the data stays stale in cache — it will
   be refetched the next time a component mounts and subscribes.
```

**Explanation:** `invalidateQueries` does not delete cached data. It marks it stale and triggers a refetch for active subscribers. This is why the UI does not flicker to empty — the old todo list is shown while the background refetch runs, then it updates with the server's confirmed list including the new item.

</details>

---

### Q19

```jsx
queryClient.setQueryData(['user', 1], (old) => ({
  ...old,
  name: 'Alice Smith',
}));
```

#### ❓ What is the difference between `setQueryData` and `invalidateQueries`? When would you use each?

<details>
<summary>✅ Answer</summary>

```txt
setQueryData: manually write data into the cache. No network request.
              Use when you already have the new data (e.g., server response
              from a mutation included updated user).

invalidateQueries: mark data stale and trigger a refetch.
                   Use when you know the server state changed but don't
                   have the new data in hand.
```

**Explanation:** `setQueryData` is synchronous and avoids a network round-trip. It is ideal for optimistic updates or when the mutation response contains the full updated entity. `invalidateQueries` is used when the mutation response only confirms success but does not return the updated data. Using `setQueryData` incorrectly with partial data can cause stale UI; using `invalidateQueries` when you have the data wastes bandwidth.

</details>

---

### Q20

```jsx
const { data: todos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos,
  select: (data) => data.filter(todo => !todo.completed),
});
```

#### ❓ The server returns 100 todos, 80 of which are completed. What does `todos` contain? Does `select` affect what is stored in cache?

<details>
<summary>✅ Answer</summary>

```txt
`todos` contains the 20 incomplete todos (the filtered result).
The cache stores the full 100 todos — `select` does NOT modify the cache.
```

**Explanation:** `select` is a client-side transformation applied after data is returned from cache. It does not affect what is stored in the cache entry. If another component queries `['todos']` without a `select`, it receives all 100 todos. `select` runs every time the cached data updates, and React Query memoizes the selected result — it only re-renders the component if the selected data actually changes.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function ProductPage() {
  const { data: product } = useQuery({
    queryKey: ['product', productId],
    queryFn: () => fetchProduct(productId),
  });

  const { data: reviews } = useQuery({
    queryKey: ['reviews', product?.id],
    queryFn: () => fetchReviews(product.id),
    enabled: !!product,
  });
}
```

#### ❓ What happens on the initial render before `product` loads? What is the role of `enabled`?

<details>
<summary>✅ Answer</summary>

```txt
On initial render: product is undefined. enabled: !!product is false.
The reviews query is disabled — no fetch fires for reviews.
When product loads: enabled becomes true, the reviews query fires.
```

**Explanation:** `enabled: false` prevents a query from running. This is the correct pattern for dependent (serial) queries. Without `enabled`, the reviews query would fire with `product.id` being `undefined`, likely causing a 400 error or fetching the wrong resource. The `!!product` guard ensures reviews are only fetched once the prerequisite data is available.

</details>

---

### Q22

```jsx
function useInfinitePostsFeed() {
  return useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam = 1 }) => fetchPosts(pageParam),
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  });
}

const { data, fetchNextPage, hasNextPage } = useInfinitePostsFeed();

// data.pages = [
//   { posts: [...20 items], nextCursor: 'cursor_21' },
//   { posts: [...20 items], nextCursor: 'cursor_41' },
// ]
```

#### ❓ How do you get a flat array of all loaded posts?

<details>
<summary>✅ Answer</summary>

```txt
const allPosts = data.pages.flatMap(page => page.posts);
```

**Explanation:** `useInfiniteQuery` stores each page as a separate entry in `data.pages`. Each element in `pages` is the raw response from one `queryFn` call. To render a flat list, you must flatten the pages array. `flatMap` is preferred over `reduce` or nested maps for readability. `hasNextPage` is `true` when `getNextPageParam` returns a non-undefined value.

</details>

---

### Q23

```jsx
const [page, setPage] = useState(1);

const { data, isPlaceholderData } = useQuery({
  queryKey: ['todos', page],
  queryFn: () => fetchTodos(page),
  placeholderData: keepPreviousData,
});

return (
  <div>
    {data?.todos.map(todo => <div key={todo.id}>{todo.text}</div>)}
    <button
      onClick={() => setPage(p => p + 1)}
      disabled={isPlaceholderData || !data?.hasMore}
    >
      Next
    </button>
  </div>
);
```

#### ❓ The user clicks "Next" to go from page 1 to page 2. What renders while page 2 loads?

<details>
<summary>✅ Answer</summary>

```txt
Page 1's data remains visible while page 2 loads.
isPlaceholderData is true during the transition — the "Next" button is disabled.
When page 2 data arrives, isPlaceholderData becomes false and the new data renders.
```

**Explanation:** `placeholderData: keepPreviousData` (TanStack Query v5) tells React Query to keep the previous page's data visible when the `queryKey` changes. This prevents the UI from going blank between page transitions. `isPlaceholderData` is `true` when the currently displayed data belongs to the previous query key — useful for disabling pagination controls to prevent double-clicking while loading.

</details>

---

### Q24

```jsx
function ProfileEditor() {
  const { data: user } = useQuery({
    queryKey: ['user'],
    queryFn: fetchCurrentUser,
  });

  const [name, setName] = useState(user?.name ?? '');

  return (
    <input value={name} onChange={e => setName(e.target.value)} />
  );
}
```

#### ❓ `user` loads asynchronously. The input starts empty. After the fetch completes, does the input auto-populate with the user's name?

<details>
<summary>✅ Answer</summary>

```txt
No. The input stays empty even after user loads.
```

**Explanation:** `useState(user?.name ?? '')` only uses its argument on the first render. At first render, `user` is `undefined`, so `name` is initialized to `''`. When the query resolves and `user` updates, `useState` ignores the new initial value — state is already initialized. Fix: use `useEffect` to sync, or use an uncontrolled input with `defaultValue`, or reset the form when user loads using a `key` prop:

```jsx
<input key={user?.id} defaultValue={user?.name ?? ''} ... />
```

</details>

---

### Q25

```jsx
function UserList() {
  const { data: users = [] } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  return (
    <ul>
      {users.map(user => (
        <UserItem key={user.id} userId={user.id} />
      ))}
    </ul>
  );
}

function UserItem({ userId }) {
  const { data: details } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUserDetails(userId),
  });

  return <li>{details?.email}</li>;
}
```

#### ❓ The list contains 50 users. How many network requests fire when UserList first mounts?

<details>
<summary>✅ Answer</summary>

```txt
51 requests: 1 for the user list, then 50 individual detail requests —
one per UserItem component.
```

**Explanation:** This is the N+1 query problem. Fetching a list and then fetching details for each item individually creates N+1 total requests. Fix options: (1) include details in the list API response to avoid separate fetches, (2) batch requests using a batching endpoint `GET /api/users?ids=1,2,3...`, (3) use `useQueries` to fire all 50 in parallel with deduplication. React Query's deduplication only helps if the same `userId` appears multiple times — it does not reduce N+1 patterns inherently.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| useEffect Fetching | Q1 – Q5 | Race conditions, ignore flag, missing deps, async warning, AbortController |
| Loading / Error States | Q6 – Q10 | Multiple loading flags, fetch not rejecting on 4xx, Error Boundary async limitation, parallel loading gate, waterfall |
| React Query Behavior | Q11 – Q15 | staleTime, cache deduplication, refetchOnWindowFocus, optimistic update with rollback, isLoading vs isFetching |
| Caching and Deduplication | Q16 – Q20 | Concurrent dedup, gcTime, invalidateQueries, setQueryData vs invalidate, select transform |
| Edge Cases | Q21 – Q25 | Dependent queries with enabled, infinite query flattening, keepPreviousData, state init from async data, N+1 problem |
