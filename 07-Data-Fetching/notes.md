# React Data Fetching

## Table of Contents

1. [Data Fetching in React](#1-data-fetching-in-react)
2. [Fetch API Basics](#2-fetch-api-basics)
3. [Data Fetching with useEffect](#3-data-fetching-with-useeffect)
4. [Race Conditions](#4-race-conditions)
5. [Loading and Error States](#5-loading-and-error-states)
6. [Axios vs Fetch](#6-axios-vs-fetch)
7. [API Layer Pattern](#7-api-layer-pattern)
8. [React Query (TanStack Query) Introduction](#8-react-query-tanstack-query-introduction)
9. [useQuery](#9-usequery)
10. [useMutation](#10-usemutation)
11. [React Query Cache](#11-react-query-cache)
12. [Optimistic Updates](#12-optimistic-updates)
13. [Pagination](#13-pagination)
14. [Infinite Queries](#14-infinite-queries)
15. [SWR (Stale-While-Revalidate)](#15-swr-stale-while-revalidate)
16. [Error Boundaries with Data Fetching](#16-error-boundaries-with-data-fetching)
17. [Common Mistakes](#17-common-mistakes)
18. [Best Practices](#18-best-practices)

---

## 1. Data Fetching in React

### Client vs Server Data

React applications work with two fundamentally different categories of data.

**Client data** — state that lives entirely in the browser and never needs to be persisted to a server. Examples: whether a modal is open, the current value of a controlled input, a user's local UI preferences.

**Server data** — state that is owned and persisted by a remote server. Examples: user profiles, product listings, posts, analytics. Server data has characteristics that make it fundamentally harder to manage:

- It is **asynchronous** — you must wait for a network round-trip before you have the data.
- It is **shared** — other users or background processes can change it between your reads.
- It can become **stale** — the value you fetched 60 seconds ago may no longer be accurate.
- It requires **synchronization** — your local copy must periodically be reconciled with the remote truth.
- It has **ownership** — the server is the source of truth, not your React state.

### Why Data Fetching Is Complex in React

React is a UI rendering library. It deliberately does not ship with a data fetching mechanism. This leaves developers responsible for:

1. Triggering requests at the correct point in the component lifecycle.
2. Storing fetched data in component or global state.
3. Handling the three possible states of any async operation: loading, success, error.
4. Cancelling stale or superseded requests.
5. Caching responses so the same request is not repeated needlessly.
6. Revalidating cached data when it grows stale.
7. Keeping server state consistent across multiple components that display the same data.

### Core Challenges in Detail

**Loading states**

Users need immediate visual feedback during network latency. Every async operation must display a spinner, skeleton, or placeholder until data arrives. Forgetting to handle the loading state is one of the most common bugs in React applications — the component renders before data is available and crashes trying to read properties of `undefined`.

**Error handling**

Network requests fail. Servers return errors. Timeouts occur. These are not exceptional situations — they are normal operating conditions. You must catch all errors, surface meaningful messages to the user, and ideally provide a retry mechanism.

**Caching**

Making the same API call every time a component mounts wastes bandwidth and makes the UI feel slow. If a user navigates away from a page and comes back, they should see the previously loaded data instantly while a background revalidation happens silently.

**Race conditions**

If a user triggers multiple requests in rapid succession — for example, typing in a search box — the responses can arrive out of order. A response from an old, slow request can overwrite the result of the most recent request, leaving the UI in an incorrect state. This is called a race condition.

**Stale data**

Cached data expires. A user might have a browser tab open for hours. The data they are viewing could be completely outdated. The application must decide how long cached data is trusted and when to silently refresh it.

**Deduplication**

If ten components on the same page all try to fetch `/api/user/profile`, only one network request should actually be made. The results should be shared among all ten components.

---

## 2. Fetch API Basics

### The `fetch()` Function

`fetch()` is a browser-native API for making HTTP requests. It returns a `Promise` that resolves to a `Response` object.

```js
fetch('https://api.example.com/users')
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error('Network error:', error));
```

Using `async/await`:

```js
async function getUsers() {
  const response = await fetch('https://api.example.com/users');
  const data = await response.json();
  return data;
}
```

### `Response.json()` and Other Body Readers

The `Response` object has multiple methods for reading the response body. Each returns a `Promise`:

```js
response.json()        // Parse body as JSON → object
response.text()        // Read body as plain string
response.blob()        // Read body as Blob (images, files)
response.arrayBuffer() // Read body as raw binary
response.formData()    // Read body as FormData
```

Important: the body can only be read once. Calling `response.json()` after `response.text()` will throw because the stream has already been consumed.

### The Critical Gotcha: `fetch` Does Not Reject on HTTP Errors

This is one of the most tested concepts in interviews. `fetch()` only rejects its `Promise` for **network-level failures**: DNS resolution failure, no internet connection, a CORS preflight blocked by the server. It does **not** reject for HTTP error status codes.

A 404 Not Found, a 500 Internal Server Error, a 403 Forbidden — none of these cause `fetch` to throw. The `Promise` resolves successfully with a `Response` object, and it is your responsibility to check whether the response indicates an error.

```js
// This code has a subtle bug:
async function getUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    const data = await response.json(); // Runs even on 404 or 500!
    return data;
  } catch (err) {
    // This only catches network errors, not HTTP 4xx/5xx
    console.error('Network error:', err);
  }
}
```

❌ Wrong — assumes fetch throws on HTTP errors:

```js
const data = await fetch('/api/users/999').then(r => r.json());
// If the server returns 404, data will contain the error JSON body
// (e.g., { message: "User not found" }), not an exception
```

✅ Correct — always check `response.ok`:

```js
async function getUser(id) {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    // response.ok is true for status 200–299, false for everything else
    throw new Error(`HTTP error: ${response.status} ${response.statusText}`);
  }

  return response.json();
}
```

`response.ok` is a boolean shorthand for `response.status >= 200 && response.status < 300`.

### Making POST, PUT, DELETE Requests

```js
async function createUser(userData) {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
    body: JSON.stringify(userData),
  });

  if (!response.ok) {
    const errorBody = await response.json().catch(() => ({}));
    throw new Error(errorBody.message || `HTTP ${response.status}`);
  }

  return response.json();
}
```

```js
async function updateUser(id, updates) {
  const response = await fetch(`/api/users/${id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(updates),
  });
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  return response.json();
}

async function deleteUser(id) {
  const response = await fetch(`/api/users/${id}`, { method: 'DELETE' });
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  // DELETE often returns 204 No Content — no body to parse
}
```

### Fetch Options Reference

```js
fetch(url, {
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`,
  },
  body: JSON.stringify(data),          // Only for POST/PUT/PATCH, not GET
  signal: abortController.signal,      // For request cancellation
  credentials: 'include',              // Send cookies cross-origin
  cache: 'no-cache' | 'force-cache',  // Cache policy
  mode: 'cors' | 'same-origin' | 'no-cors',
  redirect: 'follow' | 'error' | 'manual',
});
```

---

## 3. Data Fetching with useEffect

### The Basic Pattern

The most common raw pattern for data fetching in React uses `useEffect` combined with `useState` for the three states of any async operation.

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    setError(null);

    fetch(`/api/users/${userId}`)
      .then(response => {
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response.json();
      })
      .then(data => {
        setUser(data);
        setLoading(false);
      })
      .catch(err => {
        setError(err.message);
        setLoading(false);
      });
  }, [userId]); // Re-fetch when userId changes

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return null;

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}
```

### How useEffect Fits the Lifecycle

```txt
Component mounts
    ↓
useEffect runs (after paint)
    ↓
fetch() is called
    ↓
Component renders with loading=true
    ↓
Response arrives
    ↓
setUser() + setLoading(false) called
    ↓
Component re-renders with data
```

The effect runs *after* the first render, which means the component always renders once with `loading=true` before data is available. This is by design — `useEffect` never runs synchronously during rendering.

### Why This Pattern Has Problems

Despite being common, the manual `useEffect` pattern has significant limitations:

1. **No cleanup** — If the component unmounts before the fetch completes, the `.then()` callbacks still fire and call `setUser` on an unmounted component. React 18 suppresses the warning, but this is still a memory leak and can cause subtle bugs.

2. **No race condition handling** — Changing `userId` quickly fires multiple requests. The last response to *arrive* wins, not the last request *sent*. Results can appear out of order.

3. **No caching** — Every component mount fetches fresh data. Navigating back to a page refetches everything.

4. **No deduplication** — Two instances of `UserProfile` with the same `userId` make two separate network requests.

5. **Boilerplate explosion** — Every component that fetches data repeats the same three `useState` calls plus `useEffect` plus three conditional renders.

### Custom Hook: Encapsulating the Pattern

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) {
      setLoading(false);
      return;
    }

    let cancelled = false;
    setLoading(true);
    setError(null);

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => {
      cancelled = true;
    };
  }, [url]);

  return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <div>{user.name}</div>;
}
```

### The Dependency Array

```jsx
// Fetch once on mount only
useEffect(() => { /* fetch */ }, []);

// Re-fetch whenever userId changes
useEffect(() => { /* fetch */ }, [userId]);

// Re-fetch on every render — almost always a mistake
useEffect(() => { /* fetch */ });

// Missing dependency — silent bug
const [query, setQuery] = useState('');
useEffect(() => {
  fetchSearch(query); // query changes but effect never re-runs!
}, []); // ❌ query should be in the array
```

ESLint rule `react-hooks/exhaustive-deps` catches missing dependencies automatically. Always follow it.

---

## 4. Race Conditions

### The Problem Illustrated

Consider a search component. The user types quickly:

```txt
User types: "r"     → Request A dispatched (slow server, takes 300ms)
User types: "re"    → Request B dispatched (takes 150ms)
User types: "rea"   → Request C dispatched (takes 100ms)

Network responses arrive:
  Request C resolves (100ms) → UI shows "rea" results ← correct
  Request B resolves (150ms) → UI shows "re" results  ← stale, overwrites
  Request A resolves (300ms) → UI shows "r" results   ← very stale, overwrites again!

Final state: UI displays results for "r" even though the user typed "rea"
```

This is a race condition: the final UI state depends on arbitrary network timing rather than the logical order of user actions.

### Fix 1: AbortController

`AbortController` is the modern, correct solution. When the effect re-runs or the component unmounts, the previous request is actively cancelled by the browser.

```jsx
useEffect(() => {
  const controller = new AbortController();

  setLoading(true);
  setError(null);

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(res => {
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      return res.json();
    })
    .then(data => {
      setResults(data);
      setLoading(false);
    })
    .catch(err => {
      if (err.name === 'AbortError') {
        // This is expected — the effect cleanup cancelled the request
        // Do NOT update state here
        return;
      }
      setError(err.message);
      setLoading(false);
    });

  // Cleanup: runs when query changes (before next effect) or on unmount
  return () => {
    controller.abort();
  };
}, [query]);
```

When the effect cleanup runs (because `query` changed), `controller.abort()` is called. The browser cancels the in-flight request. The `fetch` Promise rejects with an `AbortError`, which we catch and silently ignore.

### Fix 2: Boolean Ignore Flag

When `AbortController` is not available or when working with libraries that do not accept an `AbortSignal`:

```jsx
useEffect(() => {
  let isStale = false;

  setLoading(true);

  fetchSearch(query)
    .then(data => {
      if (!isStale) {
        setResults(data);
        setLoading(false);
      }
      // If isStale is true, silently discard the response
    })
    .catch(err => {
      if (!isStale) {
        setError(err.message);
        setLoading(false);
      }
    });

  return () => {
    isStale = true; // Mark this effect's request as stale on cleanup
  };
}, [query]);
```

The network request still completes (wasting bandwidth), but the response is discarded. This achieves correctness without active cancellation.

### Comparison

| Aspect | AbortController | Boolean Flag |
|---|---|---|
| Cancels network request | ✅ Yes | ❌ No |
| Saves bandwidth | ✅ Yes | ❌ No |
| Browser support | All modern browsers | Universal |
| Works with Axios | ✅ `{ signal }` | ✅ Always |
| Works with any async fn | ❌ Only if fn accepts signal | ✅ Always |
| Code complexity | Low | Low |

Prefer `AbortController` in production. Use the boolean flag as a fallback.

---

## 5. Loading and Error States

### The Three States of Any Async Operation

Every async operation can only be in one of these states at any given moment:

```txt
idle    → No request has been initiated
loading → A request is in flight
success → The request completed successfully
error   → The request failed
```

### Anti-Pattern: Multiple Boolean Flags

```jsx
// ❌ Wrong — separate boolean flags for each state
const [isLoading, setIsLoading] = useState(false);
const [isError, setIsError] = useState(false);
const [isSuccess, setIsSuccess] = useState(false);

// Problem: these combinations are logically impossible but technically achievable:
// isLoading=true AND isError=true
// isLoading=true AND isSuccess=true
// isError=true AND isSuccess=true

// One missed setState call can leave the component in an impossible state
```

### Correct Pattern: Status Enum

```jsx
// ✅ Correct — a single status string enforces mutual exclusivity
const [status, setStatus] = useState('idle');
const [data, setData] = useState(null);
const [error, setError] = useState(null);

useEffect(() => {
  setStatus('loading');
  setError(null);

  fetchUser(userId)
    .then(user => {
      setData(user);
      setStatus('success');
    })
    .catch(err => {
      setError(err.message);
      setStatus('error');
    });
}, [userId]);

// Exhaustive rendering — every state is handled
if (status === 'idle') return <div>Enter a user ID to search</div>;
if (status === 'loading') return <Spinner />;
if (status === 'error') return <ErrorMessage message={error} />;
// At this point, TypeScript (or logic) guarantees status === 'success'
return <UserProfile data={data} />;
```

### Using a Reducer for Complex Async State

```jsx
const initialState = { status: 'idle', data: null, error: null };

function fetchReducer(state, action) {
  switch (action.type) {
    case 'FETCH_START':
      return { ...state, status: 'loading', error: null };
    case 'FETCH_SUCCESS':
      return { status: 'success', data: action.payload, error: null };
    case 'FETCH_ERROR':
      return { ...state, status: 'error', error: action.payload };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
}

function UserProfile({ userId }) {
  const [state, dispatch] = useReducer(fetchReducer, initialState);

  useEffect(() => {
    dispatch({ type: 'FETCH_START' });
    fetchUser(userId)
      .then(data => dispatch({ type: 'FETCH_SUCCESS', payload: data }))
      .catch(err => dispatch({ type: 'FETCH_ERROR', payload: err.message }));
  }, [userId]);

  // ... render based on state.status
}
```

### TypeScript Discriminated Union for Status

```tsx
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function UserProfile({ userId }: { userId: string }) {
  const [state, setState] = useState<FetchState<User>>({ status: 'idle' });

  useEffect(() => {
    setState({ status: 'loading' });
    fetchUser(userId)
      .then(data => setState({ status: 'success', data }))
      .catch(err => setState({ status: 'error', error: err.message }));
  }, [userId]);

  if (state.status === 'idle') return null;
  if (state.status === 'loading') return <Spinner />;
  if (state.status === 'error') return <div>{state.error}</div>;
  // TypeScript narrows: state.data is guaranteed to exist here
  return <div>{state.data.name}</div>;
}
```

### Displaying Stale Data During Refresh

When the user navigates back to a page they have visited, show the previous data immediately while a background refresh runs:

```jsx
const [data, setData] = useState(null);
const [isFetching, setIsFetching] = useState(false);

useEffect(() => {
  setIsFetching(true);
  fetchUser(userId)
    .then(user => setData(user))
    .finally(() => setIsFetching(false));
}, [userId]);

// Show data immediately if available, with a subtle refresh indicator
return (
  <div style={{ opacity: isFetching && !data ? 1 : isFetching ? 0.6 : 1 }}>
    {!data ? <Spinner /> : <UserProfile user={data} />}
    {isFetching && data && <RefreshIndicator />}
  </div>
);
```

---

## 6. Axios vs Fetch

### What Is Axios?

Axios is an HTTP client library that historically wrapped `XMLHttpRequest` and now also supports `fetch`. It provides a cleaner API and several features that raw `fetch` lacks.

### Automatic JSON Parsing

```js
// fetch — two-step process
const response = await fetch('/api/users');
const data = await response.json();

// axios — single step, response.data is already parsed
const { data } = await axios.get('/api/users');
```

### HTTP Error Handling

```js
// fetch — must manually check response.ok
const response = await fetch('/api/users/999');
if (!response.ok) {
  const body = await response.json();
  throw new Error(body.message || `HTTP ${response.status}`);
}
const data = await response.json();

// axios — throws automatically for 4xx and 5xx
try {
  const { data } = await axios.get('/api/users/999');
} catch (err) {
  if (axios.isAxiosError(err)) {
    console.log(err.response.status);      // 404
    console.log(err.response.data);        // Server error body
    console.log(err.response.headers);     // Response headers
  }
}
```

### Interceptors

Axios interceptors are middleware functions that run before every request is sent or after every response arrives. They are the killer feature for production applications.

```js
// Request interceptor — automatically attach auth token
axios.interceptors.request.use(
  config => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error)
);

// Response interceptor — globally handle 401
axios.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      localStorage.removeItem('accessToken');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

### Request Cancellation

```js
// Modern approach with AbortController (works in both fetch and axios)
const controller = new AbortController();

axios.get('/api/search', { signal: controller.signal });
// or
fetch('/api/search', { signal: controller.signal });

// Cancel the request
controller.abort();

// Axios-specific CancelToken (deprecated but still appears in legacy code and interviews)
const source = axios.CancelToken.source();
axios.get('/api/search', { cancelToken: source.token });
source.cancel('Operation cancelled by user');
```

### Comparison Table

| Feature | `fetch` | `axios` |
|---|---|---|
| Built-in (no install) | ✅ Yes | ❌ npm install required |
| Automatic JSON parsing | ❌ Manual `response.json()` | ✅ Yes |
| Throws on 4xx/5xx | ❌ No — must check `response.ok` | ✅ Yes |
| Request interceptors | ❌ No | ✅ Yes |
| Response interceptors | ❌ No | ✅ Yes |
| Upload progress events | ❌ No | ✅ Yes |
| Request cancellation | ✅ AbortController | ✅ AbortController + CancelToken |
| XSRF protection | ❌ No | ✅ Built-in |
| Automatic timeout | ❌ No | ✅ `{ timeout: 5000 }` |
| Browser support | Modern browsers | All (XHR fallback) |
| Bundle size impact | 0 bytes | ~13kb gzipped |

### When to Choose Each

Use `fetch` when:
- Zero dependencies is a hard requirement.
- The request is a one-off in a script or utility function.
- You are in a Service Worker context.

Use `axios` when:
- You need interceptors (almost every production React app needs this).
- You want consistent error handling across all HTTP status codes.
- You need upload progress events.
- You are building an API layer that will be used across many components.

---

## 7. API Layer Pattern

### The Problem with Scattered API Calls

When `fetch` calls are scattered throughout components:

- The base URL is duplicated dozens of times.
- Auth headers must be added manually to every request.
- There is no single place to handle token expiry.
- No easy way to swap the API implementation for tests.
- Changing the API base URL requires touching many files.

### Recommended Directory Structure

```txt
src/
  services/
    api.js           ← Axios instance + interceptors
    userService.js   ← User-related endpoints
    postService.js   ← Post-related endpoints
    authService.js   ← Auth endpoints
    index.js         ← Re-exports for convenience
  hooks/
    useUsers.js      ← React Query hooks for users
    usePosts.js      ← React Query hooks for posts
```

### The Axios Instance

```js
// src/services/api.js
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_URL || 'http://localhost:3001/api',
  timeout: 15000,
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  },
});

export default api;
```

### Request Interceptor

```js
// src/services/api.js (continued)
api.interceptors.request.use(
  config => {
    const token = localStorage.getItem('accessToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => {
    return Promise.reject(error);
  }
);
```

### Response Interceptor with Error Normalization

```js
class ApiError extends Error {
  constructor(message, status, data) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.data = data;
  }
}

api.interceptors.response.use(
  // Unwrap the data layer automatically: response.data → data
  response => response.data,

  error => {
    if (error.response) {
      // Server responded with a non-2xx status
      const { status, data } = error.response;

      if (status === 401) {
        return handleTokenRefresh(error);
      }

      if (status === 403) {
        window.location.href = '/access-denied';
        return Promise.reject(error);
      }

      if (status === 422) {
        // Validation errors — pass through for forms to handle
        return Promise.reject(
          new ApiError(data.message, status, data.errors)
        );
      }

      throw new ApiError(
        data?.message || 'Server error',
        status,
        data
      );
    }

    if (error.request) {
      // No response received (network error, timeout)
      throw new ApiError('Network error. Please check your connection.', 0, null);
    }

    // Request setup error
    throw new ApiError(error.message, -1, null);
  }
);
```

### Service Layer

```js
// src/services/userService.js
import api from './api';

export const userService = {
  getAll: (params) =>
    api.get('/users', { params }),

  getById: (id) =>
    api.get(`/users/${id}`),

  create: (userData) =>
    api.post('/users', userData),

  update: (id, userData) =>
    api.put(`/users/${id}`, userData),

  patch: (id, updates) =>
    api.patch(`/users/${id}`, updates),

  delete: (id) =>
    api.delete(`/users/${id}`),

  search: (query, signal) =>
    api.get('/users/search', {
      params: { q: query },
      signal,
    }),
};
```

### Token Refresh Pattern

This is an advanced pattern that appears in senior-level interviews. When a 401 is received, try to refresh the access token and replay the original request.

```js
let isRefreshing = false;
let failedQueue = []; // Queue of requests waiting for token refresh

function processQueue(error, token = null) {
  failedQueue.forEach(({ resolve, reject }) => {
    if (error) {
      reject(error);
    } else {
      resolve(token);
    }
  });
  failedQueue = [];
}

async function handleTokenRefresh(originalError) {
  const originalRequest = originalError.config;

  if (isRefreshing) {
    // Another request is already refreshing the token.
    // Queue this request to retry after the token is refreshed.
    return new Promise((resolve, reject) => {
      failedQueue.push({ resolve, reject });
    }).then(token => {
      originalRequest.headers.Authorization = `Bearer ${token}`;
      return api(originalRequest);
    });
  }

  isRefreshing = true;

  try {
    const refreshToken = localStorage.getItem('refreshToken');
    const { accessToken } = await axios.post('/auth/refresh', { refreshToken });

    localStorage.setItem('accessToken', accessToken);
    api.defaults.headers.Authorization = `Bearer ${accessToken}`;

    processQueue(null, accessToken);

    originalRequest.headers.Authorization = `Bearer ${accessToken}`;
    return api(originalRequest);
  } catch (refreshError) {
    processQueue(refreshError, null);
    localStorage.clear();
    window.location.href = '/login';
    return Promise.reject(refreshError);
  } finally {
    isRefreshing = false;
  }
}
```

---

## 8. React Query (TanStack Query) Introduction

### Why React Query Exists

The manual `useEffect` approach requires every team to solve the same set of problems from scratch: caching, background refetching, loading states, error states, race conditions, deduplication, optimistic updates. React Query (rebranded as TanStack Query to support multiple frameworks) provides battle-tested solutions for all of these.

React Query describes itself as a **server state management library**, which is distinct from client state management libraries like Redux or Zustand.

### Core Mental Model

React Query maintains a **cache** — a dictionary keyed by `queryKey` arrays. Each entry in the cache holds:

- The data (if fetched)
- The status (pending, success, error)
- A timestamp of when it was last updated
- Whether it is currently stale

```txt
QueryClient Cache
  ├── ["users"]                          → { data: [...], updatedAt: 1703001234 }
  ├── ["user", 42]                       → { data: {...}, updatedAt: 1703001200 }
  ├── ["posts", { category: "tech" }]   → { status: "loading" }
  └── ["user", 42, "posts"]             → { data: [...], updatedAt: 1703001100 }
```

### Installation

```bash
npm install @tanstack/react-query
# Optional but strongly recommended for development
npm install @tanstack/react-query-devtools
```

### Provider Setup

```jsx
// src/main.jsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // Data fresh for 5 minutes
      gcTime: 10 * 60 * 1000,      // Cache entries live 10 min after unmount
      retry: 2,                     // Retry failed requests twice
      retryDelay: attempt => Math.min(1000 * 2 ** attempt, 30000), // Exponential backoff
      refetchOnWindowFocus: true,   // Refetch on tab focus
      refetchOnReconnect: true,     // Refetch when network reconnects
    },
  },
});

function Root() {
  return (
    <QueryClientProvider client={queryClient}>
      <App />
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

### Glossary

| Term | Definition |
|---|---|
| `QueryClient` | The central store. Holds all cached query data. |
| `QueryClientProvider` | React context provider. Makes `QueryClient` available to all components. |
| `queryKey` | Array uniquely identifying a piece of server data. Changes trigger re-fetch. |
| `queryFn` | Async function that fetches the data. Must return a Promise. |
| `staleTime` | Duration after which fetched data is considered stale (default: 0ms). |
| `gcTime` | Duration after which unused cache entries are garbage collected (default: 5min). |
| `Mutation` | An operation that writes to the server: POST, PUT, PATCH, DELETE. |

---

## 9. useQuery

### Basic Syntax

```jsx
import { useQuery } from '@tanstack/react-query';
import { userService } from '../services/userService';

function UserList() {
  const {
    data,
    isLoading,
    isError,
    error,
    isFetching,
    isStale,
    refetch,
    status,
  } = useQuery({
    queryKey: ['users'],
    queryFn: () => userService.getAll(),
  });

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage message={error.message} />;

  return (
    <div>
      {isFetching && <RefreshBanner />}
      <ul>
        {data.map(user => <li key={user.id}>{user.name}</li>)}
      </ul>
      <button onClick={() => refetch()}>Refresh</button>
    </div>
  );
}
```

### Query Keys

The query key is the most important concept in React Query. It must uniquely identify the data.

```jsx
// Static — global data, no parameters
useQuery({ queryKey: ['config'], queryFn: fetchConfig });

// Dynamic — scoped to a specific resource
useQuery({ queryKey: ['user', userId], queryFn: () => fetchUser(userId) });

// Parameterized — key changes when filters change
useQuery({
  queryKey: ['users', { status: filter, page: pageNumber }],
  queryFn: () => fetchUsers({ status: filter, page: pageNumber }),
});

// Nested — related data
useQuery({ queryKey: ['user', userId, 'posts'], queryFn: () => fetchUserPosts(userId) });
```

When any part of the array changes, React Query treats it as a different cache entry and triggers a fresh fetch. This is the mechanism behind automatic re-fetching when parameters change.

### `isLoading` vs `isFetching` — Crucial Interview Distinction

```txt
isLoading  = isPending AND isFetching
           = There is no cached data AND a fetch is in progress
           = True only on the very first load

isFetching = A fetch is currently in progress (regardless of cached data)
           = True on first load AND on background re-fetches
```

```jsx
const { data, isLoading, isFetching } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });

// On first load: isLoading=true, isFetching=true → show full-page spinner
// On background refetch: isLoading=false, isFetching=true → show subtle indicator
// When fresh: isLoading=false, isFetching=false → show nothing

if (isLoading) return <FullPageSpinner />; // Only on cold start

return (
  <div>
    {isFetching && <SmallSpinner />} {/* Background refresh indicator */}
    <UserList users={data} />
  </div>
);
```

### Dependent Queries

```jsx
function UserPosts({ userId }) {
  // Query 1: fetch the user
  const { data: user } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Query 2: fetch the user's posts — only runs when user.id is available
  const { data: posts } = useQuery({
    queryKey: ['posts', user?.id],
    queryFn: () => fetchPostsByUser(user.id),
    enabled: Boolean(user?.id), // ← Disabled until user is loaded
  });

  return <PostList posts={posts ?? []} />;
}
```

### Configuration Options

```jsx
useQuery({
  queryKey: ['users'],
  queryFn: fetchUsers,

  // Staleness and caching
  staleTime: 5 * 60 * 1000,
  gcTime: 10 * 60 * 1000,

  // Refetch behavior
  refetchOnWindowFocus: true,
  refetchOnReconnect: true,
  refetchInterval: 30000,          // Poll every 30 seconds
  refetchIntervalInBackground: false,

  // Error handling
  retry: 3,
  retryDelay: 1000,

  // State
  enabled: Boolean(userId),        // Conditionally enable/disable
  placeholderData: keepPreviousData, // Show previous data while loading new

  // Callbacks
  select: data => data.users,      // Transform/filter data
  onSuccess: data => console.log(data), // v4 only (removed in v5)
});
```

### Full Return Value Reference

```jsx
const {
  data,              // Resolved data (undefined if no data yet)
  error,             // Error object if query failed (null otherwise)
  status,            // 'pending' | 'error' | 'success'
  fetchStatus,       // 'fetching' | 'paused' | 'idle'
  isLoading,         // true when no data AND fetching (first load)
  isPending,         // true when no data in cache (v5)
  isFetching,        // true when any fetch is in progress
  isRefetching,      // true when background re-fetch is in progress
  isSuccess,         // true when status === 'success'
  isError,           // true when status === 'error'
  isStale,           // true when data is older than staleTime
  isPlaceholderData, // true when showing keepPreviousData
  dataUpdatedAt,     // Timestamp of last successful fetch
  errorUpdatedAt,    // Timestamp of last error
  failureCount,      // Number of consecutive failures
  refetch,           // () => Promise — manually trigger a fetch
} = useQuery({ queryKey, queryFn });
```

---

## 10. useMutation

### Overview

`useQuery` is for reading data (GET requests). `useMutation` is for writing data — POST, PUT, PATCH, and DELETE operations. The key difference is that mutations are imperative (you call `mutate()` explicitly) rather than declarative.

### Basic Example

```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';

function CreateUserForm() {
  const queryClient = useQueryClient();

  const createUser = useMutation({
    mutationFn: (userData) => userService.create(userData),

    onSuccess: (newUser, variables, context) => {
      // newUser = server response
      // variables = what was passed to mutate()
      // context = what onMutate returned
      queryClient.invalidateQueries({ queryKey: ['users'] });
      toast.success(`User ${newUser.name} created`);
    },

    onError: (error, variables, context) => {
      console.error('Mutation failed:', error);
      toast.error(error.message);
    },

    onSettled: (data, error, variables, context) => {
      // Always runs after onSuccess or onError
    },
  });

  const handleSubmit = async (e) => {
    e.preventDefault();
    const data = Object.fromEntries(new FormData(e.target));
    createUser.mutate(data);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" placeholder="Name" required />
      <input name="email" type="email" placeholder="Email" required />
      <button type="submit" disabled={createUser.isPending}>
        {createUser.isPending ? 'Creating...' : 'Create User'}
      </button>
      {createUser.isError && (
        <p style={{ color: 'red' }}>{createUser.error.message}</p>
      )}
    </form>
  );
}
```

### `mutate` vs `mutateAsync`

```jsx
// mutate — fire and forget
// Uses callbacks (onSuccess, onError)
// Does not throw if mutation fails
createUser.mutate(userData);

// mutateAsync — returns a Promise
// Allows async/await with try/catch
// WILL throw if mutation fails — must wrap in try/catch
try {
  const newUser = await createUser.mutateAsync(userData);
  navigate(`/users/${newUser.id}`);
} catch (error) {
  // Handle error from mutation
}
```

### Mutation State Machine

```txt
Initial state: status=idle, isPending=false

mutate() called
    ↓
status=pending, isPending=true

    ↙               ↘
 Success            Error
    ↓                 ↓
status=success   status=error
onSuccess fires  onError fires
    ↘               ↙
    onSettled fires
```

### Invalidating Queries After Mutation

The most common pattern: after writing data to the server, mark the relevant cached queries as stale so they refetch:

```jsx
onSuccess: () => {
  // Invalidate specific query
  queryClient.invalidateQueries({ queryKey: ['users'] });

  // Invalidate all queries that start with 'users' (prefix matching)
  queryClient.invalidateQueries({ queryKey: ['users'], exact: false });

  // Invalidate multiple query types at once
  queryClient.invalidateQueries({ queryKey: ['users'] });
  queryClient.invalidateQueries({ queryKey: ['stats'] });
}
```

---

## 11. React Query Cache

### How the Cache Works

React Query stores all fetched data in an in-memory cache managed by `QueryClient`. Each cache entry is identified by a `queryKey` and contains:

- The resolved data.
- The timestamp when it was last updated (`dataUpdatedAt`).
- The current status.
- Observer count (how many components are currently subscribed).

### Cache Lifecycle

```txt
Query first requested
    ↓
No cache entry → status: 'pending'
    ↓
queryFn resolves → data stored with timestamp
status: 'success', isStale: false (fresh)
    ↓
staleTime elapses
    ↓
isStale: true
(data still shown; background re-fetch triggered on next access)
    ↓
All components using this query unmount
    ↓
gcTime countdown begins
    ↓
gcTime elapses → cache entry deleted
```

### staleTime

`staleTime` defines how long freshly fetched data is trusted before it is marked as stale.

```jsx
// Default: 0 — data is immediately stale after every fetch
// Any window focus, component mount, or manual refetch triggers a background update
useQuery({ queryKey: ['feed'], queryFn: fetchFeed, staleTime: 0 });

// 5 minutes — no background refetch for 5 minutes after last fetch
useQuery({ queryKey: ['profile'], queryFn: fetchProfile, staleTime: 5 * 60 * 1000 });

// Infinity — never stale. Fetch once, use forever (until manually invalidated)
// Good for: configuration, static lookup data, user preferences
useQuery({ queryKey: ['config'], queryFn: fetchConfig, staleTime: Infinity });
```

Data being **stale** does not mean it is removed from cache or hidden from the UI. Stale data is still shown immediately. The stale flag only indicates that a background re-fetch should occur at the next opportunity.

### gcTime (Garbage Collection Time)

`gcTime` controls how long unused data stays in the cache after all components that use it have unmounted.

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 5 * 60 * 1000, // Default: 5 minutes
    },
  },
});
```

| `gcTime` value | Effect |
|---|---|
| `0` | Data removed from cache immediately on unmount |
| `5 * 60 * 1000` (default) | Data lives 5 minutes after last subscriber unmounts |
| `Infinity` | Data never garbage collected (risk of memory leak) |

The typical pattern: `gcTime > staleTime`. This allows stale data to be shown instantly when a user navigates back, while background re-fetches keep the data fresh.

### Automatic Refetch Triggers

React Query automatically re-fetches stale data when:

1. A component subscribing to the query mounts (and data is stale).
2. The browser window regains focus (`refetchOnWindowFocus: true`).
3. The device's network connection is restored (`refetchOnReconnect: true`).
4. `queryClient.invalidateQueries()` is called.
5. The `refetchInterval` timer fires (polling).

### Manual Cache Manipulation

```jsx
const queryClient = useQueryClient();

// Invalidate: mark as stale + trigger background refetch if query is active
queryClient.invalidateQueries({ queryKey: ['users'] });

// Invalidate all queries whose key starts with 'users'
queryClient.invalidateQueries({ queryKey: ['users'], exact: false });

// Directly set cache data (bypass fetching entirely)
queryClient.setQueryData(['user', userId], updatedUser);

// Functionally update cache data
queryClient.setQueryData(['users'], old => [...old, newUser]);

// Read from cache without subscribing to changes
const cachedUser = queryClient.getQueryData(['user', userId]);

// Prefetch (fetch before navigation)
await queryClient.prefetchQuery({
  queryKey: ['user', userId],
  queryFn: () => fetchUser(userId),
});

// Remove a specific cache entry immediately
queryClient.removeQueries({ queryKey: ['user', userId] });
```

---

## 12. Optimistic Updates

### The Concept

An optimistic update immediately applies a change to the UI before the server has confirmed it, under the assumption (optimism) that the server will accept the change. If the server rejects it, the UI rolls back to its previous state.

This makes the application feel instantaneous: a user clicking a "Like" button sees the count increment immediately rather than waiting 200ms for the server response.

### Implementation with React Query

```jsx
function TodoItem({ todo }) {
  const queryClient = useQueryClient();

  const toggleTodo = useMutation({
    mutationFn: (updatedTodo) => todoService.update(updatedTodo.id, updatedTodo),

    // Step 1: Optimistically update the cache BEFORE the request fires
    onMutate: async (updatedTodo) => {
      // Cancel any in-flight refetches to prevent them from overwriting
      // our optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // Snapshot the current cache value so we can roll back on error
      const previousTodos = queryClient.getQueryData(['todos']);

      // Apply the optimistic update
      queryClient.setQueryData(['todos'], old =>
        old.map(t => t.id === updatedTodo.id ? updatedTodo : t)
      );

      // Return the snapshot as context — it will be passed to onError
      return { previousTodos };
    },

    // Step 2: If the server rejects, roll back to the snapshot
    onError: (err, updatedTodo, context) => {
      queryClient.setQueryData(['todos'], context.previousTodos);
      toast.error('Failed to update todo');
    },

    // Step 3: Always sync with the server's authoritative response
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });

  return (
    <label>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() =>
          toggleTodo.mutate({ ...todo, completed: !todo.completed })
        }
      />
      {todo.title}
    </label>
  );
}
```

### Why `cancelQueries` Is Critical

Without `cancelQueries`, the following race condition can occur:

```txt
User clicks "Like" button
  → onMutate fires, sets likeCount to 43 (optimistic)
  → Background refetch was in progress, arrives 50ms later
  → Background refetch overwrites likeCount back to 42
  → PATCH /likes completes 100ms later
  → No UI update (no invalidation triggered yet)
  → Final UI shows 42 instead of 43
```

`cancelQueries` cancels the background refetch, preventing it from overwriting the optimistic value.

---

## 13. Pagination

### Page-Based Pagination with keepPreviousData

```jsx
import { useQuery, keepPreviousData } from '@tanstack/react-query';

function UserList() {
  const [page, setPage] = useState(1);
  const PAGE_SIZE = 10;

  const { data, isLoading, isFetching, isPlaceholderData } = useQuery({
    queryKey: ['users', page],
    queryFn: () => userService.getAll({ page, limit: PAGE_SIZE }),
    placeholderData: keepPreviousData,
    // While page 2 loads, page 1 data is shown with isPlaceholderData=true
  });

  const totalPages = Math.ceil((data?.total ?? 0) / PAGE_SIZE);

  return (
    <div>
      {isLoading ? (
        <FullSpinner />
      ) : (
        <div style={{ opacity: isPlaceholderData ? 0.5 : 1 }}>
          <ul>
            {data.users.map(user => (
              <li key={user.id}>{user.name}</li>
            ))}
          </ul>
        </div>
      )}

      <nav>
        <button
          onClick={() => setPage(p => Math.max(1, p - 1))}
          disabled={page === 1 || isFetching}
        >
          Previous
        </button>

        <span>Page {page} of {totalPages}</span>

        <button
          onClick={() => setPage(p => Math.min(totalPages, p + 1))}
          disabled={page === totalPages || isFetching}
        >
          Next
        </button>
      </nav>

      {isFetching && <span>Loading...</span>}
    </div>
  );
}
```

### Prefetching the Next Page

```jsx
const queryClient = useQueryClient();

// When the user is on page 1, prefetch page 2 in the background
useEffect(() => {
  if (page < totalPages) {
    queryClient.prefetchQuery({
      queryKey: ['users', page + 1],
      queryFn: () => userService.getAll({ page: page + 1, limit: PAGE_SIZE }),
    });
  }
}, [page, totalPages, queryClient]);
```

---

## 14. Infinite Queries

### useInfiniteQuery for Feeds and Load More

```jsx
import { useInfiniteQuery } from '@tanstack/react-query';

function PostFeed() {
  const {
    data,
    fetchNextPage,
    fetchPreviousPage,
    hasNextPage,
    hasPreviousPage,
    isFetchingNextPage,
    isLoading,
    isError,
  } = useInfiniteQuery({
    queryKey: ['posts'],
    queryFn: ({ pageParam }) => postService.getAll({ cursor: pageParam, limit: 20 }),
    initialPageParam: null,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
    getPreviousPageParam: (firstPage) => firstPage.prevCursor ?? undefined,
  });

  const allPosts = data?.pages.flatMap(page => page.posts) ?? [];

  if (isLoading) return <Spinner />;
  if (isError) return <ErrorMessage />;

  return (
    <div>
      {allPosts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}

      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
        >
          {isFetchingNextPage ? 'Loading more...' : 'Load more'}
        </button>
      )}

      {!hasNextPage && <p>No more posts.</p>}
    </div>
  );
}
```

### Intersection Observer for Automatic Infinite Scroll

```jsx
function PostFeed() {
  const { fetchNextPage, hasNextPage, isFetchingNextPage, ...query } = useInfiniteQuery({ /* ... */ });
  const bottomRef = useRef(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      entries => {
        const [entry] = entries;
        if (entry.isIntersecting && hasNextPage && !isFetchingNextPage) {
          fetchNextPage();
        }
      },
      { threshold: 0.1 }
    );

    const el = bottomRef.current;
    if (el) observer.observe(el);

    return () => {
      if (el) observer.unobserve(el);
    };
  }, [fetchNextPage, hasNextPage, isFetchingNextPage]);

  const allPosts = query.data?.pages.flatMap(p => p.posts) ?? [];

  return (
    <div>
      {allPosts.map(post => <PostCard key={post.id} post={post} />)}
      <div ref={bottomRef} style={{ height: 1 }} />
      {isFetchingNextPage && <Spinner />}
    </div>
  );
}
```

---

## 15. SWR (Stale-While-Revalidate)

### What Is SWR?

SWR (stale-while-revalidate) is a data fetching library by Vercel that implements the HTTP stale-while-revalidate caching directive:

```txt
1. Return cached (stale) data immediately → fast initial render
2. Send a background request to revalidate → fresh data on the way
3. Update the UI when fresh data arrives → up-to-date without full reload
```

### Basic Usage

```bash
npm install swr
```

```jsx
import useSWR from 'swr';

const fetcher = (url) => fetch(url).then(r => {
  if (!r.ok) throw new Error(`HTTP ${r.status}`);
  return r.json();
});

function UserProfile({ id }) {
  const { data, error, isLoading } = useSWR(`/api/users/${id}`, fetcher);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage message={error.message} />;

  return <div>{data.name}</div>;
}
```

### SWR vs React Query Feature Matrix

| Feature | SWR | React Query (TanStack) |
|---|---|---|
| Gzipped bundle size | ~4kb | ~13kb |
| API simplicity | Simpler | More verbose, more options |
| Mutations | Manual (no built-in helper) | Full `useMutation` API |
| Infinite queries | `useSWRInfinite` | `useInfiniteQuery` |
| Optimistic updates | Manual | Built-in `onMutate` + rollback |
| DevTools | ❌ Not included | ✅ React Query DevTools |
| Pagination (keepPreviousData) | Manual | `placeholderData: keepPreviousData` |
| Background re-fetch | ✅ | ✅ |
| Cache invalidation | `mutate(key)` | `invalidateQueries({ queryKey })` |
| Request deduplication | ✅ | ✅ |
| Window focus re-fetch | ✅ | ✅ |
| Offline support | Basic | ✅ `networkMode` option |
| Prefetching | Manual | `prefetchQuery` |

**Recommendation:** Use React Query for any non-trivial production application. Use SWR for simple cases where bundle size is a critical constraint or where the simpler API is preferred.

---

## 16. Error Boundaries with Data Fetching

### Suspense for Declarative Loading States

React 18 + React Query v5 support Suspense mode, which lets components declare that they need data and rely on a `<Suspense>` boundary to handle the loading state.

```jsx
import { useSuspenseQuery } from '@tanstack/react-query';

// No more isLoading checks inside the component
function UserProfile({ userId }) {
  const { data } = useSuspenseQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  // Guaranteed: data is defined here
  return <div>{data.name}</div>;
}

// Suspense boundary handles loading state
function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <UserProfile userId={42} />
    </Suspense>
  );
}
```

### Error Boundaries for Data Fetch Errors

When `throwOnError: true` (or with `useSuspenseQuery`), React Query throws errors to the nearest Error Boundary.

```jsx
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log to error reporting service
    errorReporter.captureException(error, { extra: errorInfo });
  }

  render() {
    if (this.state.hasError) {
      if (this.props.fallback) return this.props.fallback;
      return (
        <div>
          <h2>Something went wrong.</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage: nest Suspense inside ErrorBoundary
function Dashboard() {
  return (
    <ErrorBoundary fallback={<ErrorPage />}>
      <Suspense fallback={<DashboardSkeleton />}>
        <DashboardContent />
      </Suspense>
    </ErrorBoundary>
  );
}
```

---

## 17. Common Mistakes

### Fetching Inside the Render Body

❌ Wrong — side effects directly in the component function:

```jsx
function UserList() {
  const [users, setUsers] = useState([]);
  // This runs on EVERY render and creates infinite re-renders
  fetch('/api/users').then(r => r.json()).then(data => setUsers(data));
  return <ul>{users.map(u => <li>{u.name}</li>)}</ul>;
}
```

✅ Correct — inside useEffect with empty dependency array:

```jsx
function UserList() {
  const [users, setUsers] = useState([]);
  useEffect(() => {
    fetch('/api/users').then(r => r.json()).then(setUsers);
  }, []);
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### Not Handling All Three States

❌ Wrong — reading `data` before it has loaded causes a crash:

```jsx
function UserList() {
  const { data } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });
  return <ul>{data.map(u => <li>{u.name}</li>)}</ul>;
  // Crashes: Cannot read properties of undefined (reading 'map')
}
```

✅ Correct — handle loading and error states explicitly:

```jsx
function UserList() {
  const { data, isLoading, isError, error } = useQuery({ ... });
  if (isLoading) return <Spinner />;
  if (isError) return <p>Error: {error.message}</p>;
  return <ul>{data.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### Missing useEffect Cleanup

❌ Wrong — state update on unmounted component:

```jsx
useEffect(() => {
  fetch(`/api/user/${id}`)
    .then(r => r.json())
    .then(data => setUser(data)); // Runs even if component unmounted
}, [id]);
```

✅ Correct — cleanup with AbortController:

```jsx
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/user/${id}`, { signal: controller.signal })
    .then(r => r.json())
    .then(data => setUser(data))
    .catch(err => { if (err.name !== 'AbortError') setError(err.message); });
  return () => controller.abort();
}, [id]);
```

### Using Array Index as React Key in Fetched Lists

❌ Wrong — using array index can cause wrong component reuse:

```jsx
{users.map((user, index) => <UserCard key={index} user={user} />)}
```

✅ Correct — use a stable, unique identifier from the server:

```jsx
{users.map(user => <UserCard key={user.id} user={user} />)}
```

### Not Checking response.ok

❌ Wrong:

```jsx
const data = await fetch('/api/users/999').then(r => r.json());
// data is {error: "User not found"} but no exception was thrown
```

✅ Correct:

```jsx
const response = await fetch('/api/users/999');
if (!response.ok) throw new Error(`HTTP ${response.status}`);
const data = await response.json();
```

---

## 18. Best Practices

### Architecture Decisions

1. **Use React Query for any non-trivial data fetching.** Implement the manual `useEffect` pattern only for demos, tutorials, or very simple applications. React Query's caching, deduplication, and background re-fetching are worth the dependency.

2. **Create an API service layer.** All `fetch` or `axios` calls should live in service files (`userService.js`, etc.), not inside components or hooks. Components should call service functions, not raw `fetch`.

3. **Use an Axios instance with interceptors.** Configure auth headers, error normalization, and token refresh once in an Axios instance. Every component benefits automatically.

4. **Separate query hooks from components.** Create custom hooks that encapsulate `useQuery` calls:

```jsx
// hooks/useUsers.js
export function useUsers(filters) {
  return useQuery({
    queryKey: ['users', filters],
    queryFn: () => userService.getAll(filters),
    staleTime: 5 * 60 * 1000,
  });
}

// components/UserList.jsx
function UserList({ filters }) {
  const { data, isLoading } = useUsers(filters);
  // ...
}
```

### Cache Configuration

5. **Set meaningful staleTime per query.** Static data (feature flags, config, enums) can use `staleTime: Infinity`. User-generated content that changes frequently can use `staleTime: 0` or `staleTime: 30000`.

6. **Invalidate cache after mutations.** After a POST/PUT/DELETE, always call `queryClient.invalidateQueries()` to ensure related queries refetch.

7. **Use `placeholderData: keepPreviousData` for paginated queries.** This eliminates the jarring "data disappears, spinner shows, data reappears" UX during page navigation.

### Error Handling

8. **Always check `response.ok` when using raw fetch.** Never assume a resolved `fetch` promise means the request succeeded.

9. **Use error boundaries around data-fetching sections.** Combine React Query's `throwOnError: true` with React error boundaries for declarative error handling.

10. **Provide retry and refresh mechanisms.** Expose a `refetch` button on error states. React Query's automatic retry with exponential backoff handles transient failures.

### Performance

11. **Prefetch data ahead of navigation.** On hover over navigation links, call `queryClient.prefetchQuery` so data is ready when the user arrives.

12. **Deduplicate at the query key level.** Multiple components using the same query key share one request automatically. Design your query keys to maximize this benefit.

### Summary Reference

| Concern | Recommended Solution |
|---|---|
| Server state management | React Query (TanStack Query) |
| API configuration | Axios instance with interceptors |
| Race conditions | `AbortController` in cleanup function |
| Loading/error states | Status enum, not multiple booleans |
| HTTP error detection | `response.ok` check |
| Error display | React Error Boundaries |
| Cache invalidation | `queryClient.invalidateQueries()` |
| Optimistic updates | `onMutate` + snapshot + `onError` rollback |
| Infinite scroll | `useInfiniteQuery` + `IntersectionObserver` |
| Pagination UX | `placeholderData: keepPreviousData` |
| Token refresh | Response interceptor with request queue |
| Prefetching | `queryClient.prefetchQuery` |
