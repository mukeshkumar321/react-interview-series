## TanStack Query — Tricky Output Questions

> These questions test deep understanding of TanStack Query's caching model, state machine, mutation lifecycle, and query invalidation. Each question is designed to reflect real interview scenarios.

---

## 1. useQuery Behavior

### Q1

```jsx
import { useQuery } from '@tanstack/react-query';

function UserCard({ userId }) {
  const { data, isPending, isFetching } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
    staleTime: 60_000,
  });

  console.log({ isPending, isFetching });
  return <div>{data?.name}</div>;
}
```

The component mounts for the first time (no cache). What is logged?

#### ❓ What are the values of `isPending` and `isFetching` on the initial render?

<details>
<summary>✅ Answer</summary>

```txt
{ isPending: true, isFetching: true }
```

**Explanation:** On the initial render, there is no data in the cache. `isPending` is `true` because the query is in `'pending'` status (no data yet). `isFetching` is `true` because an active network request is in progress. Both are `true` simultaneously during the first load. After the query resolves, `isPending` becomes `false` and `isFetching` also becomes `false`.

</details>

---

### Q2

```jsx
function UserCard({ userId }) {
  const { data, isPending, isFetching } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 0,  // immediately stale
  });

  console.log({ isPending, isFetching });
  return <div>{data?.name}</div>;
}
```

The query has previously loaded successfully and data is in the cache. The user returns to this tab (window focus event fires). What are the values logged on the background refetch?

#### ❓ What does `isPending` and `isFetching` show during the background refetch?

<details>
<summary>✅ Answer</summary>

```txt
{ isPending: false, isFetching: true }
```

**Explanation:** `isPending` is `false` because there IS data in the cache — the component is not in the initial loading state. `isFetching` is `true` because a background network request is in progress (triggered by window focus + staleTime: 0 means the data is already stale). This distinction is crucial for UX: show a subtle "refreshing" indicator via `isFetching` while displaying the cached data, rather than a full loading skeleton.

</details>

---

### Q3

```jsx
function FailingQuery() {
  const { isError, error } = useQuery({
    queryKey: ['broken'],
    queryFn: () => Promise.reject(new Error('API down')),
    retry: 2,
    retryDelay: 0,
  });

  if (isError) console.log('Error:', error.message);
  return null;
}
```

#### ❓ How many times does `queryFn` execute before `isError` becomes `true`?

<details>
<summary>✅ Answer</summary>

```txt
3 times total.
- 1 initial attempt
- 2 retries (retry: 2)

After the 3rd failure, isError becomes true and logs: Error: API down
```

**Explanation:** `retry: 2` means 2 retry attempts after the initial failure. Total executions = 1 (initial) + 2 (retries) = 3. Only after all retry attempts are exhausted does the query enter the `'error'` status and `isError` becomes `true`. The default retry count is 3 (meaning 4 total attempts). Each retry uses the `retryDelay` — here `0` so they fire immediately.

</details>

---

### Q4

```jsx
function ConditionalFetch({ shouldFetch }) {
  const { isPending, data } = useQuery({
    queryKey: ['data'],
    queryFn: fetchData,
    enabled: shouldFetch,
  });

  console.log({ isPending, data });
  return null;
}

// Rendered with:
<ConditionalFetch shouldFetch={false} />
```

#### ❓ What does `isPending` equal when `enabled: false`?

<details>
<summary>✅ Answer</summary>

```txt
{ isPending: true, data: undefined }
```

**Explanation:** When `enabled: false`, the query is in `'pending'` status because it has never been fetched — there is no data in the cache. However, `isFetching` is `false` (no network request is in progress). In v5, this can be confusing: `isPending` does not mean "loading" here, it means "no data available." To distinguish, check `isPending && !isFetching` for the "disabled but no data" case. The data is `undefined` because the queryFn has never run.

</details>

---

### Q5

```jsx
function Posts() {
  const { data } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
    select: (data) => data.filter(p => p.published),
  });

  console.log(data);
  return null;
}
```

The API returns `[{ id: 1, published: true }, { id: 2, published: false }, { id: 3, published: true }]`.

#### ❓ What does `data` contain?

<details>
<summary>✅ Answer</summary>

```txt
[{ id: 1, published: true }, { id: 3, published: true }]
```

**Explanation:** The `select` option transforms the data before it is returned to the component. The raw API response is stored in the cache, but the component receives the filtered result. `select` also provides memoization — if the raw data doesn't change, the `select` function won't rerun and the component won't re-render. The cache stores the original array; `select` is applied per-subscriber.

</details>

---

## 2. Cache and Stale

### Q6

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 5000 },
  },
});

function Component() {
  const { data, isFetching } = useQuery({
    queryKey: ['config'],
    queryFn: fetchConfig,
    // No staleTime specified here
  });
  return null;
}
```

The query loads successfully. The component unmounts and remounts 2 seconds later.

#### ❓ Does the query refetch on remount?

<details>
<summary>✅ Answer</summary>

```txt
No — the query does NOT refetch on remount.
Data is served from cache and isFetching is false.
```

**Explanation:** The `staleTime: 5000` is set globally in `QueryClient` defaults. The individual query inherits this setting. Data loaded 2 seconds ago is still within the 5-second stale window, so it is considered "fresh." When the component remounts, TanStack Query serves the cached data immediately without making a network request. A refetch would only trigger if 5+ seconds had passed since the last successful fetch.

</details>

---

### Q7

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { gcTime: 3000 },  // 3 seconds
  },
});

function App() {
  const [show, setShow] = useState(true);
  return (
    <>
      {show && <DataComponent />}
      <button onClick={() => setShow(false)}>Hide</button>
    </>
  );
}

function DataComponent() {
  const { data } = useQuery({
    queryKey: ['items'],
    queryFn: fetchItems,
  });
  return <div>{data?.length} items</div>;
}
```

`DataComponent` loads data successfully. The "Hide" button is clicked. 5 seconds pass. `DataComponent` is shown again.

#### ❓ What happens when `DataComponent` remounts after 5 seconds?

<details>
<summary>✅ Answer</summary>

```txt
The query enters 'pending' state and shows a loading state.
A fresh network request is made — the cache entry was garbage collected.
```

**Explanation:** `gcTime: 3000` means the query data is removed from cache 3 seconds after the last subscriber (component) unmounts. After 5 seconds — longer than gcTime — the cache entry is gone. When `DataComponent` remounts, TanStack Query finds no cache entry for `['items']` and starts a fresh fetch. The component shows `isPending: true`. If gcTime were longer (e.g., 10s), the data would still be in cache and the component would show the cached data immediately.

</details>

---

### Q8

```jsx
function ComponentA() {
  const { data } = useQuery({
    queryKey: ['shared'],
    queryFn: fetchSharedData,
  });
  return <div>A: {data?.value}</div>;
}

function ComponentB() {
  const { data } = useQuery({
    queryKey: ['shared'],
    queryFn: fetchSharedData,
  });
  return <div>B: {data?.value}</div>;
}

function App() {
  return (
    <>
      <ComponentA />
      <ComponentB />
    </>
  );
}
```

#### ❓ How many network requests are made when `App` first mounts?

<details>
<summary>✅ Answer</summary>

```txt
1 network request — not 2.
```

**Explanation:** TanStack Query deduplicates queries with the same key. Both `ComponentA` and `ComponentB` use `queryKey: ['shared']`. When both mount simultaneously, TanStack Query recognizes they reference the same query, fires the fetch once, and shares the result with both components. This is a core feature: multiple components can subscribe to the same data without triggering duplicate requests. Both components re-render with the same `data` when the single request resolves.

</details>

---

### Q9

```jsx
function SearchResults({ query }) {
  const { data, isPending } = useQuery({
    queryKey: ['search', query],
    queryFn: () => searchApi(query),
    placeholderData: (previousData) => previousData,  // keepPreviousData pattern
  });

  return (
    <div>
      {isPending && <Spinner />}
      {data?.results.map(r => <Result key={r.id} result={r} />)}
    </div>
  );
}
```

User types "react" — results load. User then types "react hooks". What does the component show during the fetch for "react hooks"?

#### ❓ What is displayed while fetching results for the new query?

<details>
<summary>✅ Answer</summary>

```txt
The "react" results are still displayed (placeholderData keeps previous data visible).
isPending is false (data is available from placeholder).
isFetching is true (new request in progress).
```

**Explanation:** `placeholderData: (previousData) => previousData` is the v5 equivalent of `keepPreviousData`. When the query key changes to `['search', 'react hooks']`, the new query is `'pending'` but `placeholderData` provides the previous data as a stand-in. The component continues showing the "react" results with `isFetching: true` until the new results arrive. This prevents a jarring "flash of loading state" between searches.

</details>

---

### Q10

```jsx
const queryClient = new QueryClient();

// Query loaded and cached at t=0
queryClient.setQueryData(['posts'], [{ id: 1, title: 'Hello' }]);

// 10 minutes pass — gcTime default is 5 minutes

// Check state
const data = queryClient.getQueryData(['posts']);
console.log(data);
```

#### ❓ What does `queryClient.getQueryData(['posts'])` return after 10 minutes with no component subscribed?

<details>
<summary>✅ Answer</summary>

```txt
undefined
```

**Explanation:** The default `gcTime` is 5 minutes (300,000ms). After 10 minutes with no component subscribed to the query (zero observers), the garbage collector removes the cache entry. `getQueryData` returns `undefined` for a key that doesn't exist in the cache. Note: if a component were still mounted and subscribed, the data would be kept regardless of gcTime — garbage collection only happens when there are zero active subscribers.

</details>

---

## 3. useMutation

### Q11

```jsx
function CreateUser() {
  const queryClient = useQueryClient();

  const { mutate } = useMutation({
    mutationFn: (userData) => createUserApi(userData),
    onMutate: (variables) => {
      console.log('onMutate', variables.name);
    },
    onSuccess: (data) => {
      console.log('onSuccess', data.name);
    },
    onSettled: () => {
      console.log('onSettled');
    },
  });

  mutate({ name: 'Alice' });
  // Assume createUserApi resolves successfully with { name: 'Alice', id: 1 }
}
```

#### ❓ In what order are the callbacks logged?

<details>
<summary>✅ Answer</summary>

```txt
1. onMutate Alice
2. onSuccess Alice
3. onSettled
```

**Explanation:** The mutation lifecycle order is: `onMutate` (fires synchronously before the mutationFn, before the async call) → `mutationFn` (executes) → `onSuccess` (fires if resolved) → `onSettled` (always fires last). `onMutate` is synchronous and fires immediately when `mutate()` is called. `onError` would appear between `mutationFn` failure and `onSettled`.

</details>

---

### Q12

```jsx
function LikeButton({ postId }) {
  const queryClient = useQueryClient();

  const { mutate } = useMutation({
    mutationFn: likePost,
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: ['post', id] });
      const snapshot = queryClient.getQueryData(['post', id]);
      queryClient.setQueryData(['post', id], (old) => ({
        ...old,
        likes: old.likes + 1,
      }));
      return { snapshot };
    },
    onError: (err, id, context) => {
      queryClient.setQueryData(['post', id], context.snapshot);
      console.log('Rolled back to:', context.snapshot.likes);
    },
    onSettled: (data, err, id) => {
      queryClient.invalidateQueries({ queryKey: ['post', id] });
    },
  });

  // Initial cache: { id: 1, likes: 5 }
  mutate(1);
  // Then the API call fails with a network error
}
```

#### ❓ What is the final `likes` value after the failed mutation, and what does `onError` log?

<details>
<summary>✅ Answer</summary>

```txt
Logs: "Rolled back to: 5"
Final cache after onSettled invalidation: depends on what the server returns,
but the rollback restores { likes: 5 } before invalidation triggers a fresh fetch.
```

**Explanation:** `onMutate` optimistically set `likes` to 6 and saved the snapshot `{ likes: 5 }` in context. When the API fails, `onError` receives the context and rolls back by calling `setQueryData` with the snapshot. The `console.log` prints `5` (the original value). Then `onSettled` runs and invalidates the query, which will trigger a fresh network request to get the true server state.

</details>

---

### Q13

```jsx
const { mutate, isSuccess, isPending, data } = useMutation({
  mutationFn: createPost,
});

// At render time, before any mutation has been called
console.log({ isSuccess, isPending, data });
```

#### ❓ What are the initial values of `isSuccess`, `isPending`, and `data` before `mutate()` is called?

<details>
<summary>✅ Answer</summary>

```txt
{ isSuccess: false, isPending: false, data: undefined }
```

**Explanation:** Unlike `useQuery`, `useMutation` does not automatically execute. It starts in `'idle'` status. `isSuccess` is `false`, `isPending` is `false`, `data` is `undefined`. The mutation only transitions out of idle when `mutate()` or `mutateAsync()` is called. After `mutate()` is called, `isPending` becomes `true`. After success, `isSuccess` becomes `true` and `data` holds the response.

</details>

---

### Q14

```jsx
const mutation = useMutation({
  mutationFn: async (value) => {
    if (value < 0) throw new Error('Negative value not allowed');
    return { value };
  },
  onError: (error) => {
    console.log('onError:', error.message);
  },
});

mutation.mutate(-5);
console.log('After mutate call:', mutation.isPending);
```

#### ❓ What is logged immediately after `mutation.mutate(-5)` is called (synchronously)?

<details>
<summary>✅ Answer</summary>

```txt
After mutate call: true
```

Then asynchronously:
```txt
onError: Negative value not allowed
```

**Explanation:** `mutate()` is non-blocking — it fires the async mutation and returns immediately. Synchronously after `mutate(-5)`, `isPending` is `true` because the mutation has started but not yet resolved/rejected. The `console.log` after `mutate()` runs before the async rejection. Then in a microtask/next tick, the mutation rejects, `onError` fires, and `isPending` becomes `false`.

</details>

---

### Q15

```jsx
async function handleSubmit() {
  try {
    const result = await mutation.mutateAsync({ title: 'New Post' });
    console.log('Success:', result.id);
  } catch (err) {
    console.log('Caught:', err.message);
  }
}

// Assume mutation.mutationFn throws new Error('Server error')
```

#### ❓ Does `onError` callback in `useMutation` still fire even though the error is caught with try/catch?

<details>
<summary>✅ Answer</summary>

```txt
Yes — onError still fires.

Output:
onError (from useMutation callback) fires first
then: "Caught: Server error" from try/catch
```

**Explanation:** `mutateAsync` throws the error AND fires the `onError` callback. Both happen. The `onError` callback in `useMutation` options always fires on mutation failure regardless of whether you use `mutate` or `mutateAsync`. When using `mutateAsync`, the error also propagates as a rejected promise, so your `try/catch` catches it too. With `mutate()` (not async), the error does NOT propagate as an unhandled rejection — it is only handled in `onError`.

</details>

---

## 4. Query Invalidation

### Q16

```jsx
function DeleteButton({ postId }) {
  const queryClient = useQueryClient();

  const { mutate } = useMutation({
    mutationFn: () => deletePostApi(postId),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },
  });

  return <button onClick={() => mutate()}>Delete</button>;
}

function PostsList() {
  const { data, isFetching } = useQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });
  console.log('isFetching:', isFetching);
  return null;
}
```

User clicks Delete. The delete API call succeeds. What happens to the `PostsList` query?

#### ❓ What does `PostsList` do after the delete mutation's `onSuccess` runs?

<details>
<summary>✅ Answer</summary>

```txt
The ['posts'] query is marked as stale and immediately refetches in the background.
PostsList logs: isFetching: true (during refetch)
Then after refetch: isFetching: false (with updated list)
```

**Explanation:** `invalidateQueries({ queryKey: ['posts'] })` marks all queries starting with `['posts']` as stale and triggers an immediate background refetch if there are active subscribers. Since `PostsList` is mounted, the query immediately refetches. `isFetching` becomes `true` during the refetch and `false` when complete. The `data` updates to the new list without the deleted post.

</details>

---

### Q17

```jsx
const queryClient = useQueryClient();

const { mutate } = useMutation({
  mutationFn: updateUser,
  onSuccess: (data, variables) => {
    // Option A
    queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
    // Option B
    queryClient.setQueryData(['user', variables.id], data);
  },
});
```

#### ❓ What is the difference between Option A (invalidateQueries) and Option B (setQueryData) in terms of network requests?

<details>
<summary>✅ Answer</summary>

```txt
Option A (invalidateQueries): Makes an additional network request to re-fetch the user
  from the server. Guarantees the cache matches the server's state exactly.

Option B (setQueryData): No additional network request. Directly sets the cache
  to the mutation response data. Faster but relies on the server returning
  the complete updated object.
```

**Explanation:** `invalidateQueries` is "pessimistic" — it trusts the server and re-fetches. `setQueryData` is "optimistic/direct" — it trusts the mutation response to be the truth. Use `setQueryData` when the mutation endpoint returns the complete updated resource (REST PUT/PATCH). Use `invalidateQueries` when the update might have server-side side effects not reflected in the response (triggers, computed fields, audit logs).

</details>

---

### Q18

```jsx
function App() {
  const queryClient = useQueryClient();
  
  const { mutate } = useMutation({
    mutationFn: createComment,
    onSuccess: () => {
      // Invalidate only comments, not the post itself
      queryClient.invalidateQueries({ queryKey: ['comments'] });
    },
  });

  return <button onClick={() => mutate({ text: 'Hello', postId: 1 })}>Comment</button>;
}
```

The following queries are in the cache:
```
['comments']              → active
['comments', 1]           → active
['comments', 1, 'replies'] → active
['post', 1]               → active
['user', 1]               → active
```

#### ❓ Which queries get invalidated?

<details>
<summary>✅ Answer</summary>

```txt
Invalidated (prefix match):
- ['comments']
- ['comments', 1]
- ['comments', 1, 'replies']

NOT invalidated:
- ['post', 1]
- ['user', 1]
```

**Explanation:** `invalidateQueries({ queryKey: ['comments'] })` invalidates all queries whose key **starts with** `['comments']`. This is a prefix/fuzzy match by default. All three comment-related queries match. `['post', 1]` and `['user', 1]` do not start with `['comments']` so they are unaffected. To invalidate an exact key only, use `{ exact: true }`.

</details>

---

### Q19

```jsx
function Component() {
  const { data } = useQuery({
    queryKey: ['items'],
    queryFn: fetchItems,
    staleTime: Infinity,  // never stale
  });

  return null;
}

// Somewhere else
queryClient.invalidateQueries({ queryKey: ['items'] });
```

#### ❓ Does `invalidateQueries` refetch the query even though `staleTime` is `Infinity`?

<details>
<summary>✅ Answer</summary>

```txt
Yes — invalidateQueries DOES trigger a refetch regardless of staleTime.
```

**Explanation:** `staleTime: Infinity` prevents automatic background refetches triggered by window focus, component mount, or intervals. But `invalidateQueries` is an explicit, imperative invalidation — it bypasses `staleTime`. It forcefully marks the query as stale and triggers an immediate refetch if there are active subscribers. This is by design: mutations can always force a refresh even if data was set to never expire automatically.

</details>

---

### Q20

```jsx
function UserProfile({ userId }) {
  const { data } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });
  return <div>{data?.name}</div>;
}

// After delete mutation succeeds:
queryClient.removeQueries({ queryKey: ['user', userId] });
```

#### ❓ What happens to `UserProfile` when `removeQueries` is called while it is mounted?

<details>
<summary>✅ Answer</summary>

```txt
The component re-enters isPending: true state and a new fetch is triggered immediately.
The data displayed disappears and a loading state is shown.
```

**Explanation:** `removeQueries` deletes the cache entry entirely — unlike `invalidateQueries` which marks stale and refetches in background. When a mounted component's query is removed, TanStack Query treats it as a brand new query with no data. The component re-enters the `'pending'` status and an automatic fetch fires immediately. This is why `removeQueries` should be used carefully with mounted components — prefer `invalidateQueries` for active queries unless you specifically want to clear the data immediately (e.g., on logout).

</details>

---

## 5. Advanced

### Q21

```jsx
function UserPosts({ userId }) {
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  const postsQuery = useQuery({
    queryKey: ['posts', 'user', userQuery.data?.id],
    queryFn: () => fetchPostsByUser(userQuery.data.id),
    enabled: !!userQuery.data?.id,
  });

  console.log({
    userPending: userQuery.isPending,
    postsPending: postsQuery.isPending,
    postsFetching: postsQuery.isFetching,
  });

  return null;
}
```

What is logged on the very first render (component just mounted, no cache)?

#### ❓ What are the initial values of all three logged values?

<details>
<summary>✅ Answer</summary>

```txt
{ userPending: true, postsPending: true, postsFetching: false }
```

**Explanation:** On first render, `userQuery` is pending (fetching user). `postsQuery` has `enabled: false` because `userQuery.data?.id` is `undefined` (user not loaded yet). When `enabled: false`, the query is in `'pending'` status (no data) but `isFetching` is `false` (no network request). So `postsPending: true` (no data in cache) and `postsFetching: false` (query is disabled, not fetching). After user loads, `enabled` becomes `true` and posts query starts fetching.

</details>

---

### Q22

```jsx
function Dashboard() {
  const q1 = useQuery({ queryKey: ['stats', 'daily'], queryFn: fetchDailyStats });
  const q2 = useQuery({ queryKey: ['stats', 'weekly'], queryFn: fetchWeeklyStats });
  const q3 = useQuery({ queryKey: ['stats', 'monthly'], queryFn: fetchMonthlyStats });

  // fetchDailyStats takes 100ms
  // fetchWeeklyStats takes 500ms
  // fetchMonthlyStats takes 300ms
}
```

#### ❓ What is the total wait time before all three queries have data, and why?

<details>
<summary>✅ Answer</summary>

```txt
~500ms — the maximum of the three, not the sum (100 + 500 + 300 = 900ms).
```

**Explanation:** Multiple `useQuery` calls in the same component fire simultaneously — they are not sequential. All three network requests start at the same time. The component waits for the slowest one (weeklyStats at 500ms). This is the key benefit of parallel queries: total time = max(individual times), not sum(individual times). If you needed them to run sequentially (because query 2 depends on query 1 result), you would use `enabled` to create dependent queries.

</details>

---

### Q23

```jsx
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage,
} = useInfiniteQuery({
  queryKey: ['items'],
  queryFn: ({ pageParam }) => fetchPage(pageParam),
  initialPageParam: 1,
  getNextPageParam: (lastPage) => lastPage.nextPage,
});

// After first page loads: data.pages = [{ items: [...], nextPage: 2 }]
// hasNextPage = true

fetchNextPage();
// While fetching page 2:
console.log({ isFetchingNextPage, hasNextPage });
```

#### ❓ What are the values of `isFetchingNextPage` and `hasNextPage` while page 2 is loading?

<details>
<summary>✅ Answer</summary>

```txt
{ isFetchingNextPage: true, hasNextPage: true }
```

**Explanation:** `isFetchingNextPage` is `true` while the next page fetch is in progress. `hasNextPage` is still `true` because it is determined by `getNextPageParam` on the LAST loaded page — and the last page still has `nextPage: 2`. Only after page 2 loads and `getNextPageParam` runs on page 2's response will `hasNextPage` update (to `false` if page 2 returns `nextPage: null`). The existing `data.pages` still only contains page 1 during the fetch.

</details>

---

### Q24

```jsx
function useUserWithPosts(userId) {
  return useQueries({
    queries: [
      {
        queryKey: ['user', userId],
        queryFn: () => fetchUser(userId),
      },
      {
        queryKey: ['posts', 'user', userId],
        queryFn: () => fetchUserPosts(userId),
      },
    ],
    combine: (results) => ({
      user: results[0].data,
      posts: results[1].data,
      isPending: results.some(r => r.isPending),
    }),
  });
}

const { user, posts, isPending } = useUserWithPosts(1);
```

#### ❓ What does the `combine` option in `useQueries` do?

<details>
<summary>✅ Answer</summary>

```txt
combine transforms the array of query results into a single merged object.
Instead of returning [queryResult1, queryResult2], it returns:
{ user: ..., posts: ..., isPending: boolean }
```

**Explanation:** By default, `useQueries` returns an array of query result objects. The `combine` option (v5) lets you transform that array into any shape you want. Here it extracts `data` from each result and computes a combined `isPending` flag. This avoids having to destructure and index into the array in your component. The combined result is memoized — it only recalculates when one of the individual query results changes.

</details>

---

### Q25

```jsx
function UserList() {
  const queryClient = useQueryClient();

  const { data: users } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  // On hover over a user item, prefetch their details
  const handleHover = (userId) => {
    queryClient.prefetchQuery({
      queryKey: ['user', userId],
      queryFn: () => fetchUser(userId),
      staleTime: 10_000,
    });
  };

  return (
    <ul>
      {users?.map(user => (
        <li key={user.id} onMouseEnter={() => handleHover(user.id)}>
          {user.name}
        </li>
      ))}
    </ul>
  );
}
```

User hovers over user 5. The data for user 5 is already in the cache from 2 seconds ago, and `staleTime` is 10 seconds.

#### ❓ Does `prefetchQuery` make a network request?

<details>
<summary>✅ Answer</summary>

```txt
No — prefetchQuery does NOT make a network request.
The cached data is still within the staleTime window (2s < 10s).
```

**Explanation:** `prefetchQuery` respects `staleTime`. If the cached data for `['user', 5]` was fetched 2 seconds ago and `staleTime` is 10 seconds, the data is still "fresh." `prefetchQuery` will skip the network call and return immediately. A network request would only be made if: (a) there is no cache entry, or (b) the cache entry exists but is older than `staleTime`. This makes hover-prefetching cheap — repeated hovers over the same item don't hammer the API.

</details>

---

## Topics Covered

| Category | Questions | Concepts Tested |
|---|---|---|
| useQuery Behavior | Q1–Q5 | isPending vs isFetching, background refetch states, retry count, enabled:false, select transform |
| Cache and Stale | Q6–Q10 | staleTime inheritance, gcTime eviction, query deduplication, placeholderData, manual setQueryData |
| useMutation | Q11–Q15 | Callback order, optimistic rollback, initial mutation state, mutate synchrony, mutateAsync error propagation |
| Query Invalidation | Q16–Q20 | invalidateQueries triggers refetch, invalidate vs setQueryData, prefix matching, staleTime:Infinity override, removeQueries on mount |
| Advanced | Q21–Q25 | Dependent query initial state, parallel query timing, isFetchingNextPage, useQueries combine, prefetchQuery respects staleTime |
