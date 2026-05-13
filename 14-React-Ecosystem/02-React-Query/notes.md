# TanStack Query (React Query)

## Table of Contents

1. [What is TanStack Query](#1-what-is-tanstack-query)
2. [Installation and Setup](#2-installation-and-setup)
3. [useQuery Hook](#3-usequery-hook)
4. [Query States](#4-query-states)
5. [staleTime and gcTime](#5-staletime-and-gctime)
6. [Background Refetching](#6-background-refetching)
7. [useMutation Hook](#7-usemutation-hook)
8. [Query Invalidation](#8-query-invalidation)
9. [Optimistic Updates](#9-optimistic-updates)
10. [Prefetching](#10-prefetching)
11. [Parallel Queries](#11-parallel-queries)
12. [Dependent Queries](#12-dependent-queries)
13. [Infinite Queries](#13-infinite-queries)
14. [Query Keys Best Practices](#14-query-keys-best-practices)
15. [React Query vs SWR vs RTK Query](#15-react-query-vs-swr-vs-rtk-query)
16. [DevTools](#16-devtools)
17. [Common Mistakes](#17-common-mistakes)
18. [Best Practices](#18-best-practices)

---

## 1. What is TanStack Query

TanStack Query (formerly React Query) is an **async state management library** for React. It specializes in managing **server state** — data that lives on a remote server and needs to be fetched, cached, synchronized, and updated.

### The Problem It Solves

Without TanStack Query, managing server data requires significant manual boilerplate:

```jsx
// ❌ Manual server state management — verbose and error-prone
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setIsLoading(true);
    setError(null);
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(data => {
        if (!cancelled) {
          setUser(data);
          setIsLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err);
          setIsLoading(false);
        }
      });
    return () => { cancelled = true; };
  }, [userId]);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <div>{user?.name}</div>;
}
```

This code has problems: no caching (re-fetches every mount), no deduplication (multiple components trigger the same fetch), no background refresh, manual cancellation, manual error handling, and no shared state between components.

```jsx
// ✅ With TanStack Query — concise, cached, shared, background-refreshed
function UserProfile({ userId }) {
  const { data: user, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <div>{user?.name}</div>;
}
```

### Server State vs Client State

| Dimension | Client State | Server State |
|---|---|---|
| Lives in | Browser memory | Remote server / database |
| Examples | Modal open, selected tab, form input | Users list, posts, orders |
| Ownership | Your app | Shared — other users can change it |
| Synchronization | Not needed | Required (can go stale) |
| Persistence | In-memory or localStorage | Database |
| Tool | useState, Zustand, Redux | TanStack Query, SWR |

### Core Features

- **Automatic caching** — responses are cached by query key
- **Background refetching** — stale data is refreshed behind the scenes
- **Deduplication** — multiple components using the same query share one request
- **Automatic retries** — failed requests are retried with exponential backoff
- **Window focus refetching** — refreshes when the user returns to the tab
- **Pagination and infinite scrolling** — built-in patterns
- **Optimistic updates** — show changes immediately, roll back on error
- **Server state synchronization** — `invalidateQueries` triggers re-fetches after mutations

### v5 Breaking Changes (TanStack Query v5)

| v4 API | v5 API |
|---|---|
| `useQuery(key, fn, options)` | `useQuery({ queryKey, queryFn, ...options })` |
| `cacheTime` | `gcTime` (garbage collection time) |
| `isLoading` for initial load | `isPending` (isLoading now means "has data but fetching") |
| `status === 'loading'` | `status === 'pending'` |

---

## 2. Installation and Setup

```bash
npm install @tanstack/react-query
npm install @tanstack/react-query-devtools  # optional but recommended
```

### QueryClient

`QueryClient` is the central cache manager. It holds all query data and manages background operations.

```ts
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// Create one QueryClient per application (outside component tree)
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,    // 5 minutes — global default
      gcTime: 1000 * 60 * 10,      // 10 minutes — global default
      retry: 2,                    // retry failed requests 2 times
      refetchOnWindowFocus: true,  // refetch when user returns to tab
    },
    mutations: {
      retry: 0,  // do not retry mutations by default
    },
  },
});
```

### QueryClientProvider

Wrap your application with `QueryClientProvider` to make the client available to all hooks:

```jsx
// main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

**Important:** Create the `QueryClient` outside the component or in a `useState` to prevent recreation on every render:

```jsx
// ✅ Correct — created outside component
const queryClient = new QueryClient();

function App() {
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>;
}

// ✅ Also correct — created with useState (useful for testing)
function App() {
  const [queryClient] = useState(() => new QueryClient());
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>;
}
```

---

## 3. useQuery Hook

`useQuery` is the primary hook for fetching and caching server data.

### Basic Usage

```jsx
import { useQuery } from '@tanstack/react-query';

function Posts() {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ['posts'],
    queryFn: async () => {
      const res = await fetch('/api/posts');
      if (!res.ok) throw new Error('Failed to fetch posts');
      return res.json();
    },
  });

  if (isPending) return <p>Loading...</p>;
  if (isError) return <p>Error: {error.message}</p>;
  return <ul>{data.map(post => <li key={post.id}>{post.title}</li>)}</ul>;
}
```

### All Key Options

```ts
const result = useQuery({
  queryKey: ['posts'],             // Required: unique identifier for caching
  queryFn: fetchPosts,             // Required: async function that returns data
  staleTime: 1000 * 60 * 5,       // How long data is "fresh" (ms) — default 0
  gcTime: 1000 * 60 * 10,         // How long unused data stays in cache (ms)
  enabled: true,                   // Set to false to disable auto-fetching
  retry: 3,                        // Number of times to retry on failure
  retryDelay: attemptIndex =>      // Custom retry delay
    Math.min(1000 * 2 ** attemptIndex, 30000),
  refetchOnWindowFocus: true,      // Refetch when window regains focus
  refetchOnMount: true,            // Refetch when component mounts
  refetchInterval: false,          // Polling interval in ms (false = disabled)
  select: (data) => data.items,    // Transform/select portion of data
  placeholderData: [],             // Data to show while loading (no loading state)
  initialData: [],                 // Treat as real data (no loading state, cached)
});
```

### Return Value

```ts
const {
  data,              // The resolved data (undefined while loading)
  error,             // Error object if query failed
  status,            // 'pending' | 'error' | 'success'
  isPending,         // true while loading for the first time (no data in cache)
  isLoading,         // true when fetching AND no cached data
  isFetching,        // true whenever a fetch is in progress (including background)
  isError,           // true if the query errored
  isSuccess,         // true if the query succeeded
  isStale,           // true if data is older than staleTime
  refetch,           // Function to manually trigger a refetch
  dataUpdatedAt,     // Timestamp of last successful fetch
} = useQuery({ ... });
```

### The `select` Option

```jsx
// Only re-renders when the selected value changes
const { data: postCount } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  select: (data) => data.length,  // component only gets the count
});

// Extract a specific item
const { data: firstPost } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  select: (data) => data[0],
});
```

---

## 4. Query States

Understanding the state machine is critical for handling all UI scenarios correctly.

### State Diagram

```text
           ┌─────────────────────────────────────────┐
           │              Status States               │
           │  pending → success                       │
           │  pending → error                         │
           │  success → (refetching) → success        │
           │  error   → (retrying)  → error/success   │
           └─────────────────────────────────────────┘

           ┌─────────────────────────────────────────┐
           │              fetchStatus                 │
           │  idle    — no fetch in progress          │
           │  fetching — fetch actively running       │
           │  paused  — fetch paused (no network)     │
           └─────────────────────────────────────────┘
```

### Status vs FetchStatus

```jsx
function Component() {
  const { data, status, fetchStatus, isPending, isFetching } = useQuery({ ... });

  // status reflects data availability
  // fetchStatus reflects network activity

  // All possible combinations:
  // status: 'pending',  fetchStatus: 'fetching'  → Initial fetch in progress
  // status: 'pending',  fetchStatus: 'paused'    → Offline, waiting to fetch
  // status: 'success',  fetchStatus: 'idle'      → Data available, no fetch
  // status: 'success',  fetchStatus: 'fetching'  → Background refetch
  // status: 'error',    fetchStatus: 'idle'      → Failed, not retrying
  // status: 'error',    fetchStatus: 'fetching'  → Retrying

  if (isPending) {
    // No data at all — show full loading state
    return <FullPageSpinner />;
  }

  if (isError) {
    return <ErrorBoundary />;
  }

  return (
    <div>
      {isFetching && <SmallRefetchIndicator />}  {/* Background refresh */}
      <DataDisplay data={data} />
    </div>
  );
}
```

### Handling All States Properly

```jsx
function UserCard({ userId }) {
  const { data, isPending, isFetching, isError, error, isStale } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 30_000,
  });

  if (isPending) return <Skeleton />;
  if (isError) return <p>Failed to load: {error.message}</p>;

  return (
    <div>
      {isFetching && <RefreshSpinner />}  {/* Shows during background refresh */}
      {isStale && <span>Stale data</span>}
      <h2>{data.name}</h2>
    </div>
  );
}
```

---

## 5. staleTime and gcTime

These two settings control the caching behavior and are frequently confused.

### staleTime — How Long Data is Fresh

`staleTime` defines how long fetched data is considered **fresh**. While data is fresh, TanStack Query will not refetch it even on component mount or window focus.

```text
Data fetched
     ↓
─────────────────────────────────────────────────
0         staleTime          gcTime (after unmount)
│←─ FRESH ────────→│←─────── STALE ─────────────→│
│  No background   │  Background refetch allowed  │
│  refetching      │  on focus/mount/interval     │
─────────────────────────────────────────────────
```

```jsx
// staleTime: 0 (default) — data is immediately stale
// Every mount, focus, or interval triggers a background refetch
const { data } = useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,
  staleTime: 0,  // immediately stale — refetch on every interaction
});

// staleTime: 5 minutes — data stays fresh for 5 minutes
const { data } = useQuery({
  queryKey: ['config'],
  queryFn: fetchConfig,
  staleTime: 1000 * 60 * 5,  // no refetch for 5 minutes
});

// staleTime: Infinity — never stale — only fetches once
const { data } = useQuery({
  queryKey: ['constants'],
  queryFn: fetchConstants,
  staleTime: Infinity,  // fetch once, never refetch automatically
});
```

### gcTime — How Long Unused Data Stays in Cache

`gcTime` (formerly `cacheTime`) defines how long **unused** (unmounted) query data stays in the cache before being garbage collected.

```text
Component using query unmounts
     ↓
─────────────────────────────────────────────────────
0                    gcTime
│←─ DATA KEPT IN CACHE ──────→│← DATA REMOVED ──────
│  Component remounts: instant │  Component remounts:
│  data from cache             │  loading state again
─────────────────────────────────────────────────────
```

```jsx
const { data } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  gcTime: 1000 * 60 * 10,  // keep in cache for 10 minutes after unmount
});
```

### Key Distinction

| | staleTime | gcTime |
|---|---|---|
| What it controls | When to background-refetch | When to remove from cache |
| Affects | Refetch behavior | Memory usage |
| Default | 0 (immediately stale) | 5 minutes (300,000ms) |
| During | While data is in use | While data is unused (component unmounted) |
| If expired | Data is refetched in background | Data is deleted from cache |

**Example: staleTime > gcTime (unusual but valid)**

```jsx
// Data stays fresh for 10 minutes but removed from cache after 1 minute of disuse
// If remounted within 1 minute: instant data from cache
// If remounted after 1 minute: loading state, fresh fetch
const { data } = useQuery({
  queryKey: ['expensive'],
  queryFn: fetchExpensiveData,
  staleTime: 1000 * 60 * 10,   // fresh for 10 minutes
  gcTime: 1000 * 60 * 1,        // removed from cache after 1 minute of disuse
});
```

---

## 6. Background Refetching

TanStack Query automatically keeps data fresh through several triggers.

### refetchOnWindowFocus

When the user switches tabs and returns, stale queries automatically refetch. This is enabled by default.

```jsx
const { data } = useQuery({
  queryKey: ['notifications'],
  queryFn: fetchNotifications,
  refetchOnWindowFocus: true,   // default — refetch on tab focus
  // refetchOnWindowFocus: false  — disable
  // refetchOnWindowFocus: 'always'  — refetch even if fresh
});
```

### refetchOnMount

Controls whether to refetch when a component mounts.

```jsx
const { data } = useQuery({
  queryKey: ['config'],
  queryFn: fetchConfig,
  refetchOnMount: true,     // default — refetch on mount if stale
  // refetchOnMount: false    — use cache, never refetch on mount
  // refetchOnMount: 'always' — always refetch on mount regardless of staleTime
});
```

### refetchInterval (Polling)

```jsx
// Poll every 5 seconds — useful for live data
const { data } = useQuery({
  queryKey: ['live-price'],
  queryFn: fetchLivePrice,
  refetchInterval: 5000,  // refetch every 5 seconds
});

// Conditional polling — stop when we have what we need
const { data } = useQuery({
  queryKey: ['job-status', jobId],
  queryFn: () => fetchJobStatus(jobId),
  refetchInterval: (query) => {
    // Stop polling when job is complete
    return query.state.data?.status === 'completed' ? false : 2000;
  },
});
```

### Manual Refetch

```jsx
const { data, refetch, isFetching } = useQuery({ ... });

return (
  <button onClick={() => refetch()} disabled={isFetching}>
    {isFetching ? 'Refreshing...' : 'Refresh'}
  </button>
);
```

---

## 7. useMutation Hook

`useMutation` handles create, update, and delete operations (server-side mutations).

### Basic Usage

```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreatePost() {
  const queryClient = useQueryClient();

  const { mutate, isPending, isError, error } = useMutation({
    mutationFn: (newPost) =>
      fetch('/api/posts', {
        method: 'POST',
        body: JSON.stringify(newPost),
        headers: { 'Content-Type': 'application/json' },
      }).then(r => r.json()),

    onSuccess: (data) => {
      // data is the resolved value from mutationFn
      console.log('Created:', data);
      queryClient.invalidateQueries({ queryKey: ['posts'] });
    },

    onError: (error) => {
      console.error('Failed:', error.message);
    },
  });

  return (
    <button
      onClick={() => mutate({ title: 'New Post', content: 'Hello' })}
      disabled={isPending}
    >
      {isPending ? 'Creating...' : 'Create Post'}
    </button>
  );
}
```

### Mutation Lifecycle Callbacks

```text
mutate() called
     ↓
onMutate(variables)     ← runs before mutation, returns context
     ↓
mutationFn(variables)   ← actual API call
     ↓
  ┌──────────────────────────────────────┐
  │ success path         │ error path    │
  │ onSuccess(data,      │ onError(err,  │
  │   vars, context)     │   vars, ctx)  │
  └──────────────────────────────────────┘
     ↓
onSettled(data, err, vars, context)   ← always runs
```

```jsx
const mutation = useMutation({
  mutationFn: updateUser,

  onMutate: async (variables) => {
    // Called immediately when mutate() is invoked
    // Used for optimistic updates — return a context object
    await queryClient.cancelQueries({ queryKey: ['user', variables.id] });
    const previousUser = queryClient.getQueryData(['user', variables.id]);
    queryClient.setQueryData(['user', variables.id], variables);
    return { previousUser };  // This becomes the `context` parameter below
  },

  onSuccess: (data, variables, context) => {
    // data: resolved value from mutationFn
    // variables: what was passed to mutate()
    // context: what onMutate returned
    console.log('Updated successfully');
  },

  onError: (error, variables, context) => {
    // Roll back optimistic update
    queryClient.setQueryData(['user', variables.id], context.previousUser);
  },

  onSettled: (data, error, variables, context) => {
    // Runs after onSuccess OR onError — always
    queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
  },
});
```

### mutate vs mutateAsync

```jsx
const { mutate, mutateAsync } = useMutation({ ... });

// mutate — fire and forget; errors handled in onError
function handleClick() {
  mutate({ title: 'Post' });  // no try/catch needed
}

// mutateAsync — returns a promise; errors must be caught
async function handleClick() {
  try {
    const result = await mutateAsync({ title: 'Post' });
    console.log('Created:', result);
  } catch (err) {
    console.error('Failed:', err);
  }
}
```

---

## 8. Query Invalidation

After a mutation, the cache is stale and needs to be refreshed. `invalidateQueries` marks queries as stale and triggers background refetches.

### Basic Invalidation

```jsx
const queryClient = useQueryClient();

// Invalidate all queries with key starting with 'posts'
queryClient.invalidateQueries({ queryKey: ['posts'] });

// Invalidate exact key only
queryClient.invalidateQueries({ queryKey: ['posts', 1], exact: true });

// Invalidate all queries
queryClient.invalidateQueries();
```

### Invalidation After Mutation

```jsx
const deletePost = useMutation({
  mutationFn: (id) => fetch(`/api/posts/${id}`, { method: 'DELETE' }),
  onSuccess: () => {
    // Invalidate the posts list — triggers refetch of posts list
    queryClient.invalidateQueries({ queryKey: ['posts'] });
    // Also invalidate any individual post queries that might be cached
    queryClient.invalidateQueries({ queryKey: ['post'] });
  },
});
```

### Manual Cache Updates (Alternative to Refetch)

```jsx
// After creating a post, update the cache directly instead of refetching
const createPost = useMutation({
  mutationFn: (newPost) => postToApi(newPost),
  onSuccess: (createdPost) => {
    queryClient.setQueryData(['posts'], (oldData) => {
      return [...(oldData ?? []), createdPost];
    });
  },
});
```

### removeQueries vs invalidateQueries

```jsx
// invalidateQueries — marks stale, triggers background refetch on next access
queryClient.invalidateQueries({ queryKey: ['posts'] });

// removeQueries — removes from cache entirely (next access shows loading)
queryClient.removeQueries({ queryKey: ['posts'] });
```

---

## 9. Optimistic Updates

Optimistic updates make the UI respond immediately to mutations, before the server confirms the change.

### Pattern

```jsx
function LikeButton({ postId }) {
  const queryClient = useQueryClient();

  const { mutate } = useMutation({
    mutationFn: (postId) =>
      fetch(`/api/posts/${postId}/like`, { method: 'POST' }).then(r => r.json()),

    onMutate: async (postId) => {
      // 1. Cancel any outgoing refetches (so they don't overwrite our optimistic update)
      await queryClient.cancelQueries({ queryKey: ['post', postId] });

      // 2. Snapshot the previous value
      const previousPost = queryClient.getQueryData(['post', postId]);

      // 3. Optimistically update the cache
      queryClient.setQueryData(['post', postId], (old) => ({
        ...old,
        likes: old.likes + 1,
        likedByMe: true,
      }));

      // 4. Return context with snapshot for rollback
      return { previousPost };
    },

    onError: (err, postId, context) => {
      // 5. On error, roll back to the snapshot
      queryClient.setQueryData(['post', postId], context.previousPost);
    },

    onSettled: (data, error, postId) => {
      // 6. Always refetch to ensure server and client are in sync
      queryClient.invalidateQueries({ queryKey: ['post', postId] });
    },
  });

  return <button onClick={() => mutate(postId)}>Like</button>;
}
```

### Optimistic Update Flow

```text
User clicks Like
     ↓
onMutate runs:
  - cancelQueries (prevent race conditions)
  - snapshot previousPost
  - setQueryData (optimistic update — UI shows +1 like immediately)
     ↓
mutationFn runs (API call in background)
     ↓
  ┌────────────────────────────────────────┐
  │ Success               │ Error          │
  │ onSettled:            │ onError:       │
  │ invalidateQueries     │ rollback from  │
  │ (confirms server data)│ snapshot       │
  └────────────────────────────────────────┘
```

---

## 10. Prefetching

Prefetching loads data into the cache before a component that needs it is rendered.

### queryClient.prefetchQuery

```jsx
import { useQueryClient } from '@tanstack/react-query';

function PostsList({ posts }) {
  const queryClient = useQueryClient();

  return (
    <ul>
      {posts.map(post => (
        <li
          key={post.id}
          // Prefetch on hover — data will be ready when user clicks
          onMouseEnter={() =>
            queryClient.prefetchQuery({
              queryKey: ['post', post.id],
              queryFn: () => fetchPost(post.id),
              staleTime: 1000 * 60,  // 1 minute — don't prefetch if fresh
            })
          }
        >
          <Link to={`/posts/${post.id}`}>{post.title}</Link>
        </li>
      ))}
    </ul>
  );
}
```

### Server-Side Prefetching (Next.js)

```tsx
// app/posts/page.tsx
import { QueryClient, dehydrate, HydrationBoundary } from '@tanstack/react-query';

export default async function PostsPage() {
  const queryClient = new QueryClient();

  // Prefetch on the server — data is dehydrated into the HTML
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  return (
    // Rehydrate on the client — no loading state, instant data
    <HydrationBoundary state={dehydrate(queryClient)}>
      <PostsComponent />
    </HydrationBoundary>
  );
}
```

---

## 11. Parallel Queries

### Multiple useQuery Calls

Multiple `useQuery` calls in the same component run in parallel automatically:

```jsx
function Dashboard() {
  const usersQuery = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  const postsQuery = useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
  const statsQuery = useQuery({ queryKey: ['stats'], queryFn: fetchStats });

  // All three fetches run simultaneously — not sequentially
  if (usersQuery.isPending || postsQuery.isPending || statsQuery.isPending) {
    return <Loading />;
  }

  return (
    <div>
      <Users data={usersQuery.data} />
      <Posts data={postsQuery.data} />
      <Stats data={statsQuery.data} />
    </div>
  );
}
```

### useQueries for Dynamic Number of Queries

When the number of queries is determined at runtime, use `useQueries`:

```jsx
import { useQueries } from '@tanstack/react-query';

function UserComparison({ userIds }) {
  const userQueries = useQueries({
    queries: userIds.map(id => ({
      queryKey: ['user', id],
      queryFn: () => fetchUser(id),
    })),
  });

  // userQueries is an array of query results
  const allLoaded = userQueries.every(q => q.isSuccess);
  const users = userQueries.map(q => q.data);

  if (!allLoaded) return <Loading />;
  return <ComparisonTable users={users} />;
}
```

---

## 12. Dependent Queries

When one query depends on data from another query, use the `enabled` option.

### Basic Dependent Query

```jsx
function UserPosts({ userId }) {
  // Step 1: Fetch the user
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Step 2: Fetch posts only after user is available
  const postsQuery = useQuery({
    queryKey: ['posts', 'by-user', userQuery.data?.id],
    queryFn: () => fetchPostsByUser(userQuery.data.id),
    enabled: !!userQuery.data?.id,  // only runs when user.id exists
  });

  if (userQuery.isPending) return <p>Loading user...</p>;
  if (postsQuery.isPending && postsQuery.isFetching) return <p>Loading posts...</p>;

  return (
    <div>
      <h1>{userQuery.data.name}'s Posts</h1>
      <ul>{postsQuery.data?.map(p => <li key={p.id}>{p.title}</li>)}</ul>
    </div>
  );
}
```

### enabled with Multiple Conditions

```jsx
const { data } = useQuery({
  queryKey: ['search', query, filters],
  queryFn: () => searchApi(query, filters),
  // Only fetch when query has at least 2 characters and filters are loaded
  enabled: query.length >= 2 && filtersLoaded,
});
```

---

## 13. Infinite Queries

`useInfiniteQuery` is designed for paginated or infinite scroll scenarios.

### Basic Setup

```jsx
import { useInfiniteQuery } from '@tanstack/react-query';

function InfinitePostsFeed() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isPending,
  } = useInfiniteQuery({
    queryKey: ['posts', 'infinite'],
    queryFn: ({ pageParam }) =>
      fetch(`/api/posts?cursor=${pageParam}&limit=10`).then(r => r.json()),
    initialPageParam: 0,  // v5: explicit initialPageParam required
    getNextPageParam: (lastPage, allPages) => {
      // Return next cursor, or undefined to indicate no more pages
      return lastPage.nextCursor ?? undefined;
    },
  });

  if (isPending) return <Loading />;

  return (
    <div>
      {data.pages.map((page, i) => (
        <Fragment key={i}>
          {page.posts.map(post => (
            <PostCard key={post.id} post={post} />
          ))}
        </Fragment>
      ))}
      <button
        onClick={() => fetchNextPage()}
        disabled={!hasNextPage || isFetchingNextPage}
      >
        {isFetchingNextPage ? 'Loading more...' : hasNextPage ? 'Load More' : 'Nothing more to load'}
      </button>
    </div>
  );
}
```

### Data Structure

```text
data.pages = [
  { posts: [...page1], nextCursor: 10 },   ← page 1
  { posts: [...page2], nextCursor: 20 },   ← page 2
  { posts: [...page3], nextCursor: null }, ← page 3 (last)
]

data.pageParams = [0, 10, 20]  ← the pageParam used for each page
```

### Intersection Observer for Auto-Load

```jsx
import { useEffect, useRef } from 'react';

function InfiniteList() {
  const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({ ... });
  const loadMoreRef = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && hasNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.1 }
    );

    if (loadMoreRef.current) observer.observe(loadMoreRef.current);
    return () => observer.disconnect();
  }, [fetchNextPage, hasNextPage]);

  return (
    <div>
      {data?.pages.flatMap(page => page.items).map(item => (
        <Item key={item.id} item={item} />
      ))}
      <div ref={loadMoreRef} />  {/* Sentinel element */}
    </div>
  );
}
```

---

## 14. Query Keys Best Practices

Query keys uniquely identify a query in the cache. They should be deterministic and descriptive.

### Always Use Arrays

```jsx
// ✅ Array keys — recommend always
useQuery({ queryKey: ['posts'], queryFn: fetchPosts });
useQuery({ queryKey: ['post', postId], queryFn: () => fetchPost(postId) });

// ❌ Avoid string keys (deprecated in v4, removed in v5 mindset)
useQuery({ queryKey: 'posts', queryFn: fetchPosts });
```

### Hierarchical Structure

```jsx
// Resource → ID → sub-resource → filters
['posts']                        // all posts
['post', postId]                 // single post
['posts', { status: 'active' }]  // filtered posts
['user', userId]                 // single user
['user', userId, 'posts']        // user's posts
['user', userId, 'posts', { page: 1 }]  // user's posts page 1
```

### Granular Keys Enable Precise Invalidation

```jsx
// Invalidate all user-related queries
queryClient.invalidateQueries({ queryKey: ['user'] });

// Only invalidate user 123's posts
queryClient.invalidateQueries({ queryKey: ['user', 123, 'posts'] });
```

### Query Key Factory Pattern

```jsx
// queries/userKeys.ts — centralized key factory
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (filters: UserFilters) => [...userKeys.lists(), filters] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: number) => [...userKeys.details(), id] as const,
  posts: (userId: number) => [...userKeys.detail(userId), 'posts'] as const,
};

// Usage
useQuery({ queryKey: userKeys.detail(userId), queryFn: () => fetchUser(userId) });
queryClient.invalidateQueries({ queryKey: userKeys.all }); // invalidates everything user-related
```

---

## 15. React Query vs SWR vs RTK Query

| Dimension | TanStack Query | SWR | RTK Query |
|---|---|---|---|
| Maintainer | TanStack | Vercel | Redux Toolkit team |
| Bundle size | ~13 kB | ~7 kB | Part of RTK (~10 kB) |
| Learning curve | Moderate | Low | Higher (requires Redux setup) |
| Mutations | Full useMutation hook | Manual (no built-in) | Full createApi endpoints |
| Optimistic updates | Built-in pattern | Manual | Built-in with updateQueryData |
| Infinite query | useInfiniteQuery | useSWRInfinite | Custom (paginated queries) |
| Prefetching | prefetchQuery | preload | prefetch via endpoint |
| DevTools | Excellent dedicated panel | None built-in | Redux DevTools |
| Backend coupling | Agnostic | Agnostic | Designed with Redux in mind |
| Offline support | Yes (fetchStatus: 'paused') | Limited | Yes |
| Next.js SSR | HydrationBoundary | SWRConfig fallback | Custom setup |
| Best for | Most React apps | Simple data fetching | Apps already using Redux |

### When to Choose What

**TanStack Query:** The default choice for React applications. Most powerful, best DX, framework-agnostic.

**SWR:** When you need a lightweight, simple solution with minimal setup and Vercel/Next.js integration.

**RTK Query:** When your app already uses Redux Toolkit and you want co-located API definitions with the rest of your Redux code.

---

## 16. DevTools

```jsx
// Install
npm install @tanstack/react-query-devtools

// main.tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <Router />
      {/* DevTools only included in development builds */}
      <ReactQueryDevtools
        initialIsOpen={false}    // collapsed by default
        buttonPosition="bottom-right"
      />
    </QueryClientProvider>
  );
}
```

### What DevTools Show

- All active queries and their current state (pending, success, error, stale)
- Query data with JSON viewer
- Query meta (staleTime, gcTime, refetch triggers)
- Active observers (how many components are subscribed)
- Ability to manually refetch, invalidate, or remove queries
- Mutations history

---

## 17. Common Mistakes

### Not Providing a Stable Query Key

```jsx
// ❌ Wrong — new object reference every render → infinite loop / cache miss
const { data } = useQuery({
  queryKey: ['posts', { filters }],  // if filters is created inline
  queryFn: fetchPosts,
});

// ✅ Correct — stable primitive values in key
const { data } = useQuery({
  queryKey: ['posts', filters.status, filters.page],
  queryFn: () => fetchPosts(filters),
});
```

### Using async/await Without Throwing on Error

```jsx
// ❌ Wrong — 4xx/5xx responses don't throw, error state never activates
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: async () => {
    const res = await fetch('/api/user');
    return res.json();  // 404 JSON would be treated as success!
  },
});

// ✅ Correct — explicitly throw on HTTP errors
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: async () => {
    const res = await fetch('/api/user');
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  },
});
```

### Putting Server State in useState

```jsx
// ❌ Wrong — duplicating server state in local state
function Component() {
  const { data } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  const [users, setUsers] = useState(data);  // stale copy, sync issues
}

// ✅ Correct — use data directly from useQuery
function Component() {
  const { data: users } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
}
```

### Creating QueryClient Inside Component

```jsx
// ❌ Wrong — new QueryClient on every render, cache lost between renders
function App() {
  const queryClient = new QueryClient();  // recreated every render!
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>;
}

// ✅ Correct — stable client reference
const queryClient = new QueryClient();
function App() {
  return <QueryClientProvider client={queryClient}>...</QueryClientProvider>;
}
```

---

## 18. Best Practices

### Use Query Key Factories

Centralize all query keys in a factory object per resource. This prevents typos and enables precise invalidation.

### Set Global Defaults Appropriately

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,  // 1 minute is a reasonable default
      retry: 1,              // retry once (default 3 is often too many for UX)
      refetchOnWindowFocus: process.env.NODE_ENV === 'production',
    },
  },
});
```

### Separate queryFn into Service Functions

```jsx
// ✅ Testable, reusable
// services/userService.ts
export const fetchUser = (id: string) =>
  fetch(`/api/users/${id}`).then(r => {
    if (!r.ok) throw new Error('Failed');
    return r.json();
  });

// components/UserCard.tsx
const { data } = useQuery({ queryKey: ['user', id], queryFn: () => fetchUser(id) });
```

### Use Suspense Mode for Cleaner Components

```jsx
// Enable suspense for a query
const { data } = useQuery({
  queryKey: ['user'],
  queryFn: fetchUser,
  suspense: true,  // throws a Promise while loading — works with React Suspense
});

// Component can be simpler — no isPending check needed
function UserCard() {
  const { data: user } = useSuspenseQuery({  // v5 dedicated hook
    queryKey: ['user'],
    queryFn: fetchUser,
  });
  return <h1>{user.name}</h1>;  // data is guaranteed defined
}

// Wrap with Suspense in parent
<Suspense fallback={<Skeleton />}>
  <UserCard />
</Suspense>
```

### Summary Reference

| Concern | Recommendation |
|---|---|
| Query keys | Use array format with hierarchical structure; use key factories |
| staleTime | Set per-query or globally; 0 is aggressive for most use cases |
| Errors | Always throw in queryFn for HTTP errors |
| Loading states | Use `isPending` for initial load; `isFetching` for background refetch |
| Mutations | Use `onSuccess` for cache updates; `onSettled` for invalidation |
| Optimistic updates | Snapshot → update → rollback on error → invalidate on settled |
| Prefetch | Prefetch on hover or in parallel with server rendering |
| DevTools | Always include in development |
| QueryClient | Create outside component or with `useState(() => new QueryClient())` |
