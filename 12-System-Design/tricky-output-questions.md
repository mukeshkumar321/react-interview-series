## 📚 React Frontend System Design — Tricky Output Questions

> These questions test your ability to reason through design decisions, predict behavior, and identify failure modes in real-world system design scenarios. Each question presents a code snippet or architectural decision and asks what happens, what is wrong, or what the best approach is.

---

## 1. Autocomplete Design

### Q1

```jsx
function Autocomplete() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  function handleChange(e) {
    setQuery(e.target.value)
    fetch(`/api/search?q=${e.target.value}`)
      .then(res => res.json())
      .then(data => setResults(data.results))
  }

  return <input value={query} onChange={handleChange} />
}
```

#### ❓ What is wrong with this implementation?

<details>
<summary>✅ Answer</summary>

```txt
Two critical problems:

1. No debounce — every single keystroke fires an API request.
   For the query "react hooks", that is 11 separate HTTP requests.
   This hammers the server and wastes bandwidth.

2. No race condition protection — if the user types "re" then "react",
   two requests fire. If the response for "re" arrives AFTER the response
   for "react", setResults will overwrite the correct results with stale data.

Fix:
- Add useDebounce(query, 300) and only fetch when debouncedQuery changes
- Use AbortController to cancel the previous request before each new fetch
```

</details>

---

### Q2

```jsx
function Autocomplete() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300)

  useEffect(() => {
    if (!debouncedQuery) return

    let cancelled = false
    fetch(`/api/search?q=${debouncedQuery}`)
      .then(res => res.json())
      .then(data => {
        if (!cancelled) setResults(data.results)
      })

    return () => { cancelled = true }
  }, [debouncedQuery])
}
```

#### ❓ Does this prevent race conditions? What is the difference between this and using AbortController?

<details>
<summary>✅ Answer</summary>

```txt
This PARTIALLY prevents the stale-data problem — it avoids setting state
from a superseded request. However, it does NOT cancel the in-flight
network request itself.

With `cancelled = true` stale flag:
- Request still runs to completion consuming bandwidth and server resources
- Only the state update is suppressed

With AbortController:
- The underlying HTTP request is aborted mid-flight (network level)
- Saves bandwidth and server compute
- The .catch() receives an AbortError which must be filtered out

For production: AbortController is strictly better.
```

</details>

---

### Q3

```jsx
function Autocomplete() {
  const [selectedIndex, setSelectedIndex] = useState(-1)
  const [suggestions, setSuggestions] = useState([])

  function handleKeyDown(e) {
    if (e.key === 'ArrowDown') {
      setSelectedIndex(selectedIndex + 1)
    }
    if (e.key === 'ArrowUp') {
      setSelectedIndex(selectedIndex - 1)
    }
  }
}
```

#### ❓ What happens when the user presses ArrowDown on the last suggestion? What about ArrowUp on the first?

<details>
<summary>✅ Answer</summary>

```txt
ArrowDown on last item: selectedIndex becomes suggestions.length,
which is out of bounds. suggestions[selectedIndex] is undefined.
Pressing Enter would select nothing and likely throw an error.

ArrowUp on first item (index 0): selectedIndex becomes -1.
Pressing ArrowUp again makes it -2, -3, etc. — unconstrained negative.

Fix — wrap around and clamp:

if (e.key === 'ArrowDown') {
  setSelectedIndex(prev =>
    prev < suggestions.length - 1 ? prev + 1 : 0  // wrap to top
  )
}
if (e.key === 'ArrowUp') {
  setSelectedIndex(prev =>
    prev > 0 ? prev - 1 : suggestions.length - 1  // wrap to bottom
  )
}
```

</details>

---

### Q4

```jsx
<input
  type="text"
  onChange={e => setQuery(e.target.value)}
  onKeyDown={handleKeyDown}
/>
{isOpen && (
  <ul>
    {suggestions.map((s, i) => (
      <li key={s.id} onClick={() => handleSelect(s)}>
        {s.label}
      </li>
    ))}
  </ul>
)}
```

#### ❓ A user clicks a suggestion but the dropdown closes before the click registers. What causes this and how do you fix it?

<details>
<summary>✅ Answer</summary>

```txt
Cause: The input's onBlur fires before the li's onClick.
When the input loses focus, setIsOpen(false) hides the dropdown,
causing the click target (the li) to be removed from the DOM
before the click event fires. The click is lost.

Fix: Use onMouseDown on the suggestion list items instead of onClick.
onMouseDown fires before onBlur, so you can call e.preventDefault()
to prevent the blur from firing:

<li
  key={s.id}
  onMouseDown={(e) => {
    e.preventDefault()   // prevents input blur
    handleSelect(s)
  }}
>
  {s.label}
</li>

Alternatively, use a short setTimeout in the onBlur:
onBlur={() => setTimeout(() => setIsOpen(false), 150)}
```

</details>

---

## 2. Infinite Scroll

### Q5

```jsx
const sentinelRef = useRef()

useEffect(() => {
  const observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting) {
      fetchNextPage()
    }
  })
  observer.observe(sentinelRef.current)
  return () => observer.disconnect()
}, [fetchNextPage])

return (
  <div>
    {posts.map(post => <PostCard key={post.id} post={post} />)}
    <div ref={sentinelRef} />
  </div>
)
```

#### ❓ This calls fetchNextPage multiple times rapidly when the sentinel enters the viewport. What is causing this and how do you fix it?

<details>
<summary>✅ Answer</summary>

```txt
Cause: fetchNextPage is called every time the IntersectionObserver
fires while the sentinel is intersecting. Because each page load
appends items, the sentinel stays in the viewport briefly during
the layout shift, potentially firing multiple times. Additionally,
fetchNextPage may change reference on each render (if not memoized),
causing the effect to re-run and re-observe with a fresh observer.

Fix 1: Gate on isFetchingNextPage to prevent concurrent fetches:

if (entry.isIntersecting && hasNextPage && !isFetchingNextPage) {
  fetchNextPage()
}

Fix 2: Use a threshold to only trigger when fully visible:

new IntersectionObserver(callback, { threshold: 1.0 })

Fix 3: Ensure fetchNextPage is stable (React Query provides this).
```

</details>

---

### Q6

```jsx
const [page, setPage] = useState(1)
const [posts, setPosts] = useState([])

async function loadMore() {
  const data = await fetchPosts(page)
  setPosts(prev => [...prev, ...data.posts])
  setPage(prev => prev + 1)
}
```

#### ❓ Why can offset-based pagination cause duplicate posts in a real-time feed?

<details>
<summary>✅ Answer</summary>

```txt
Offset pagination uses: LIMIT 20 OFFSET 0, LIMIT 20 OFFSET 20, etc.

Problem: If a new post is inserted at the top of the feed between
page 1 and page 2 requests, every existing post shifts down by 1
row in the database result set.

Timeline:
  - Page 1 fetch: rows 0-19 → returns posts A, B, C, ... T
  - New post X is inserted at position 0
  - Page 2 fetch: OFFSET 20 → returns posts T, U, V, ... (T is duplicated!)

Fix: Use cursor-based pagination.
  Page 1 returns posts and a cursor (e.g., the ID of the last post).
  Page 2 request includes: WHERE id < <cursor> ORDER BY id DESC LIMIT 20
  New insertions do not affect cursor position.
```

</details>

---

### Q7

```jsx
function Feed() {
  const [posts, setPosts] = useState([])
  const [newPosts, setNewPosts] = useState([])

  useEffect(() => {
    const ws = new WebSocket('/ws/feed')
    ws.onmessage = (e) => {
      const post = JSON.parse(e.data)
      setPosts(prev => [post, ...prev]) // prepend new post
    }
  }, [])
}
```

#### ❓ A new post is prepended via WebSocket while the user is scrolled 500px down. What UX problem occurs and how do you fix it?

<details>
<summary>✅ Answer</summary>

```txt
Problem: Prepending an item to the DOM shifts all existing content
downward. The user's scroll position stays at 500px from the top
(by pixel), which now points to a DIFFERENT post. This is jarring —
the feed appears to "jump" or the user loses their reading position.

Fix — notify, don't auto-insert:
  Store new posts in a separate newPosts array.
  Show a "X new posts" banner at the top.
  Only merge newPosts into posts when the user clicks the banner
  OR when the user has scrolled back to the top (scrollTop === 0).

function handleShowNewPosts() {
  setPosts(prev => [...newPosts, ...prev])
  setNewPosts([])
  window.scrollTo({ top: 0, behavior: 'smooth' })
}
```

</details>

---

### Q8

```jsx
useEffect(() => {
  const observer = new IntersectionObserver(([entry]) => {
    if (entry.isIntersecting && hasNextPage) {
      fetchNextPage()
    }
  }, { rootMargin: '0px' })

  observer.observe(sentinelRef.current)
  return () => observer.disconnect()
}, [hasNextPage])
```

#### ❓ The user reaches the bottom and the next page loads, but only after they see blank space. How do you preload the next page before the user reaches the bottom?

<details>
<summary>✅ Answer</summary>

```txt
rootMargin: '0px' means the sentinel must be fully in the viewport
before loading starts. By then the user already sees the end.

Fix: Use a positive rootMargin to trigger loading BEFORE the sentinel
reaches the viewport:

new IntersectionObserver(callback, {
  rootMargin: '400px'   // start loading 400px before the sentinel
})

This prefetches the next page while the user is still 400px above
the bottom, creating a seamless experience with no visible blank space.
```

</details>

---

## 3. Real-Time

### Q9

```jsx
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/ws')
  ws.onclose = () => {
    setTimeout(() => {
      setWs(new WebSocket('wss://api.example.com/ws'))
    }, 3000)
  }
  setWs(ws)
  return () => ws.close()
}, [])
```

#### ❓ What is the problem with this reconnection logic?

<details>
<summary>✅ Answer</summary>

```txt
Two problems:

1. Infinite reconnection with fixed delay.
   If the server is down, this creates a reconnect storm — the client
   fires a new connection every 3 seconds indefinitely, hammering the
   server as it comes back online.

   Fix: Use exponential backoff with jitter.
   delay = Math.min(1000 * 2^attempt, 30000) + Math.random() * 1000

2. The effect cleanup closes ws but the setTimeout fires AFTER cleanup
   and creates a new WebSocket that is not registered anywhere and will
   not be closed by React.

   Fix: Track the reconnect timeout in a ref and clear it in cleanup.

const reconnectTimeoutRef = useRef(null)

ws.onclose = () => {
  reconnectTimeoutRef.current = setTimeout(connect, getBackoffDelay())
}

return () => {
  ws.close()
  clearTimeout(reconnectTimeoutRef.current)
}
```

</details>

---

### Q10

```jsx
const { mutate: sendMessage } = useMutation({
  mutationFn: (text) => api.sendMessage(roomId, text),
  onMutate: (text) => {
    const tempMessage = { id: 'temp', text, status: 'sending' }
    setMessages(prev => [...prev, tempMessage])
  },
  onError: () => {
    setMessages(prev => prev.filter(m => m.id !== 'temp'))
  }
})
```

#### ❓ A user rapidly sends 3 messages. What is wrong with using a hardcoded id of 'temp'?

<details>
<summary>✅ Answer</summary>

```txt
All three optimistic messages share the same id: 'temp'.
This causes:

1. Key collision in the React list — React uses id as key,
   so all three map to the same DOM node. Only one message renders.

2. The onError cleanup filter removes ALL messages with id 'temp',
   even messages that succeeded.

3. When server confirmations arrive, there is no way to match
   which server-confirmed message corresponds to which temp message.

Fix: Generate a unique tempId per message:

const tempId = crypto.randomUUID()
const tempMessage = { id: tempId, text, status: 'sending' }

// In onError, target only this specific message:
setMessages(prev => prev.filter(m => m.id !== tempId))
```

</details>

---

### Q11

```jsx
function handleInputChange(e) {
  setInput(e.target.value)
  ws.send(JSON.stringify({ type: 'TYPING_START' }))
}
```

#### ❓ What happens if the component unmounts while the user is still typing?

<details>
<summary>✅ Answer</summary>

```txt
The TYPING_STOP signal is never sent to the server.

Consequences:
- Other chat participants see the typing indicator indefinitely
- The server's typing state never clears for this user
- If the server has a timer to auto-clear, it eventually clears —
  but this creates a delay-based UX inconsistency

Fix: Send TYPING_STOP in the useEffect cleanup:

useEffect(() => {
  return () => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type: 'TYPING_STOP', userId: currentUserId }))
    }
    clearTimeout(typingTimeoutRef.current)
  }
}, [])
```

</details>

---

## 4. State Design

### Q12

```js
// Shape A — array of posts
const state = {
  posts: [
    { id: '1', title: 'Post 1', authorId: 'u1', author: { id: 'u1', name: 'Alice' } },
    { id: '2', title: 'Post 2', authorId: 'u1', author: { id: 'u1', name: 'Alice' } },
  ]
}

// Shape B — normalized
const state = {
  posts: { '1': { id: '1', title: 'Post 1', authorId: 'u1' }, '2': { ... } },
  users: { 'u1': { id: 'u1', name: 'Alice' } }
}
```

#### ❓ Alice changes her display name to "Alice Smith". How many updates are needed in Shape A vs. Shape B?

<details>
<summary>✅ Answer</summary>

```txt
Shape A (denormalized array):
  Alice's name is embedded inside EVERY post she authored.
  If Alice has 50 posts, you must update 50 objects.
  This requires mapping over the entire posts array and mutating
  each post that has authorId 'u1'. This is O(n) and error-prone.

Shape B (normalized):
  Alice's name exists exactly ONCE in users['u1'].
  A single update: state.users['u1'].name = 'Alice Smith'
  All posts that reference authorId 'u1' automatically display
  the updated name. This is O(1).

Normalization is superior for write-heavy, relational data.
For read-heavy feeds with no updates, denormalization avoids
the join cost.
```

</details>

---

### Q13

```js
// Option A
const [allPosts, setAllPosts] = useState([])
const [currentPage, setCurrentPage] = useState(1)
const ITEMS_PER_PAGE = 20

const displayedPosts = allPosts.slice(
  (currentPage - 1) * ITEMS_PER_PAGE,
  currentPage * ITEMS_PER_PAGE
)

// Option B
const [pages, setPages] = useState({})   // { 1: Post[], 2: Post[] }
const [currentPage, setCurrentPage] = useState(1)

const displayedPosts = pages[currentPage] ?? []
```

#### ❓ What is wrong with Option A when using server-side pagination?

<details>
<summary>✅ Answer</summary>

```txt
Option A assumes ALL data is loaded into allPosts at once
(client-side pagination). For server-side pagination:

- You only load 20 items at a time from the server
- allPosts only ever has 20 items
- slice() always returns the same 20 items regardless of currentPage
- Navigating to page 2 shows the same page 1 data

Option A is only valid for client-side pagination where all data
is fetched upfront.

Option B correctly maps page numbers to their fetched item arrays.
On page navigation, fetch the page if not cached: pages[newPage] ?? fetchPage(newPage)
```

</details>

---

### Q14

```jsx
function SearchPage() {
  const [query, setQuery] = useState('')
  const [results, setResults] = useState([])

  // User can share the search results URL with a colleague
  // Colleague opens the URL and gets blank results
}
```

#### ❓ Why does the colleague see blank results and what is the correct state design?

<details>
<summary>✅ Answer</summary>

```txt
The query is stored in React component state, which is not
reflected in the URL. When a colleague opens the URL, the query
starts as '' and results start as [], even if the sharer had
a specific query active.

Fix: Store the search query in the URL as a query parameter.
React state should be derived FROM the URL, not vice versa.

// Read query from URL
const [searchParams, setSearchParams] = useSearchParams()
const query = searchParams.get('q') ?? ''

// Update URL when user types
function handleSearch(value) {
  setSearchParams({ q: value })
}

// Fetch based on URL state
useEffect(() => {
  if (query) fetchResults(query).then(setResults)
}, [query])

Now the URL is the source of truth. The colleague can share
/search?q=react+hooks and both see identical results.
```

</details>

---

## 5. Performance

### Q15

```jsx
function PostList({ posts }) {
  return (
    <div>
      {posts.map(post => (
        <PostCard key={post.id} post={post} />
      ))}
    </div>
  )
}
```

#### ❓ This component is given 10,000 posts. What happens in the browser and how do you fix it?

<details>
<summary>✅ Answer</summary>

```txt
10,000 DOM nodes are created and inserted.

Consequences:
1. Initial render takes seconds — creating 10,000 real DOM nodes
   is expensive (style calculation, layout, paint)
2. Scrolling is janky — the browser must maintain 10,000 nodes in memory
3. Memory usage spikes — React fiber nodes + DOM nodes for all 10,000 items
4. Any re-render of the parent forces reconciliation of all 10,000 items

Fix: Virtual scrolling — render only the items currently in the viewport
(typically 20-50 items) plus a small overscan buffer.

import { useVirtualizer } from '@tanstack/react-virtual'

const rowVirtualizer = useVirtualizer({
  count: posts.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 120,
  overscan: 5,
})

Only ~20-30 PostCard components exist in the DOM at any time,
regardless of list length.
```

</details>

---

### Q16

```jsx
function ProductPage() {
  const { data: product } = useQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id),
  })

  const { data: reviews } = useQuery({
    queryKey: ['reviews', id],
    queryFn: () => fetchReviews(id),
  })

  const { data: related } = useQuery({
    queryKey: ['related', id],
    queryFn: () => fetchRelated(id),
  })
}
```

#### ❓ These three queries fire sequentially. How do you make them fire in parallel?

<details>
<summary>✅ Answer</summary>

```txt
These queries fire in parallel already IF all three useQuery calls
execute in the same render. React Query fires all enabled queries
immediately on mount without waiting for each other.

However, if any of these queries depended on data from a previous
query (waterfall pattern), they would be sequential.

The waterfall problem looks like:
  const { data: product } = useQuery(...)
  const { data: reviews } = useQuery({
    enabled: !!product,         // WATERFALL: waits for product
    queryFn: () => fetchReviews(product.reviewIds)
  })

Fix waterfalls by restructuring APIs to return all needed data
in one request, or by using useQueries for dynamic parallel queries:

const queries = useQueries({
  queries: [
    { queryKey: ['product', id], queryFn: () => fetchProduct(id) },
    { queryKey: ['reviews', id], queryFn: () => fetchReviews(id) },
    { queryKey: ['related', id], queryFn: () => fetchRelated(id) },
  ]
})
```

</details>

---

### Q17

```jsx
function Dashboard() {
  const [data, setData] = useState(null)

  useEffect(() => {
    fetchDashboardData().then(setData)
  }, [])

  if (!data) return <Spinner />
  return <DashboardContent data={data} />
}
```

#### ❓ The dashboard shows a spinner for 2 seconds on every navigation. How do you eliminate the spinner?

<details>
<summary>✅ Answer</summary>

```txt
Cause: data starts as null on every mount, causing the spinner
to always show until the fetch resolves.

Multiple fixes:

1. React Query stale-while-revalidate:
   useQuery returns cached data instantly (if cached), shows it
   immediately, and refetches in the background. No spinner on repeat visits.

2. Prefetch on hover/focus:
   When the user hovers a nav link to dashboard, prefetch the data:
   queryClient.prefetchQuery({ queryKey: ['dashboard'], queryFn: fetchDashboard })
   By the time they click, data is already cached.

3. Route-level prefetching with React Router loaders:
   export async function loader() {
     return await fetchDashboardData()
   }
   Data is fetched BEFORE the component renders.

4. Skeleton loaders instead of spinner:
   Even without eliminating the delay, showing a skeleton layout
   that matches the content shape is significantly less jarring than
   a centered spinner. It communicates content structure early.
```

</details>

---

### Q18

```jsx
function ImageGallery({ images }) {
  return (
    <div className="gallery">
      {images.map(image => (
        <img key={image.id} src={image.url} alt={image.alt} />
      ))}
    </div>
  )
}
```

#### ❓ A gallery of 100 images causes a CLS (Cumulative Layout Shift) score of 0.8. What causes this and how do you fix it?

<details>
<summary>✅ Answer</summary>

```txt
Cause: Images without explicit width and height have 0 height
until they load. When images load, they push content down,
causing layout shifts. 100 images loading at different times
creates massive cumulative shift.

Fix 1: Always specify width and height attributes.
The browser reserves the correct space before the image loads.

<img src={image.url} alt={image.alt} width={300} height={200} />

Fix 2: Use aspect-ratio CSS to reserve space based on ratio.

.gallery-item {
  aspect-ratio: 3 / 2;
  width: 100%;
  background: #f0f0f0; /* placeholder color */
}

Fix 3: Use a blur-hash or LQIP (Low Quality Image Placeholder)
that fills the space immediately with a blurred preview.

Fix 4: Add loading="lazy" so below-fold images don't load
until needed, reducing simultaneous layout shifts.
```

</details>

---

## 6. Error Handling

### Q19

```jsx
async function handleUpload(file) {
  try {
    await uploadFile(file)
    showSuccess('Upload complete')
  } catch (error) {
    showError('Upload failed')
  }
}
```

#### ❓ The upload fails on attempt 1 due to a network timeout. Design a retry mechanism with exponential backoff.

<details>
<summary>✅ Answer</summary>

```txt
async function uploadWithRetry(file, maxAttempts = 3) {
  let lastError

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await uploadFile(file)
    } catch (error) {
      lastError = error

      // Don't retry on non-recoverable errors
      if (error.status === 400 || error.status === 413) {
        throw error
      }

      if (attempt < maxAttempts) {
        const delay = Math.min(1000 * 2 ** (attempt - 1), 10000)
        const jitter = Math.random() * 500
        await new Promise(resolve => setTimeout(resolve, delay + jitter))
      }
    }
  }

  throw lastError
}

// Delays: attempt 1 fails immediately → wait ~1s
//         attempt 2 fails → wait ~2s
//         attempt 3 fails → throw
```

</details>

---

### Q20

```jsx
async function submitBatchUpdate(updates) {
  const results = await Promise.all(
    updates.map(update => api.updateItem(update))
  )
  showSuccess(`${updates.length} items updated`)
}
```

#### ❓ What happens if 3 of 10 updates fail? How do you handle partial failure correctly?

<details>
<summary>✅ Answer</summary>

```txt
Promise.all rejects as soon as ONE promise rejects.
The other 9 updates may or may not complete — there is no way
to know which succeeded. The error handler sees only the first
error, with no information about the 9 remaining operations.

Fix: Use Promise.allSettled which waits for all promises
regardless of individual failures:

const results = await Promise.allSettled(
  updates.map(update => api.updateItem(update))
)

const succeeded = results.filter(r => r.status === 'fulfilled')
const failed = results.filter(r => r.status === 'rejected')

if (failed.length > 0) {
  showPartialError(
    `${succeeded.length} items updated, ${failed.length} failed.`,
    { retryItems: failed.map((_, i) => updates[i]) }
  )
} else {
  showSuccess(`All ${updates.length} items updated`)
}
```

</details>

---

### Q21

```jsx
function App() {
  return (
    <ErrorBoundary>
      <Header />
      <main>
        <Sidebar />
        <Feed />
        <Recommendations />
      </main>
    </ErrorBoundary>
  )
}
```

#### ❓ The Recommendations widget throws an error. What happens to the Feed? How should Error Boundaries be placed?

<details>
<summary>✅ Answer</summary>

```txt
With a single top-level ErrorBoundary, when Recommendations throws,
the ENTIRE application unmounts and shows the error fallback UI.
The user cannot use the Feed, Header, or Sidebar either.

Principle: Error Boundaries should isolate failure to the smallest
meaningful unit.

Fix: Wrap each independently renderable section:

function App() {
  return (
    <>
      <Header />   {/* no boundary needed — errors here are catastrophic */}
      <main>
        <ErrorBoundary fallback={<SidebarError />}>
          <Sidebar />
        </ErrorBoundary>
        <ErrorBoundary fallback={<FeedError />}>
          <Feed />
        </ErrorBoundary>
        <ErrorBoundary fallback={null}>  {/* silently hide broken widget */}
          <Recommendations />
        </ErrorBoundary>
      </main>
    </>
  )
}

Now a Recommendations failure shows nothing in that section,
but Feed and Sidebar continue working.
```

</details>

---

## 7. Edge Cases

### Q22

```jsx
async function handleSubmit(e) {
  e.preventDefault()
  setLoading(true)
  try {
    await submitForm(formData)
    navigate('/success')
  } finally {
    setLoading(false)
  }
}
```

#### ❓ A user double-clicks the submit button. What happens and how do you prevent it?

<details>
<summary>✅ Answer</summary>

```txt
Two form submission requests fire nearly simultaneously.
setLoading(true) is called twice — but state updates are batched
in React 18, so this may not prevent the second submission.
The server receives two identical requests and may create duplicate records.

Fix 1: Disable the button while loading:

<button type="submit" disabled={loading}>
  {loading ? 'Submitting...' : 'Submit'}
</button>

Fix 2: Use a ref guard to prevent concurrent submissions:

const submittingRef = useRef(false)

async function handleSubmit(e) {
  e.preventDefault()
  if (submittingRef.current) return   // guard
  submittingRef.current = true
  setLoading(true)
  try {
    await submitForm(formData)
    navigate('/success')
  } finally {
    submittingRef.current = false
    setLoading(false)
  }
}

Fix 3: Idempotency key — send a unique request ID with each submission.
Server ignores duplicate requests with the same idempotency key.
```

</details>

---

### Q23

```jsx
function App() {
  const [isOnline, setIsOnline] = useState(navigator.onLine)

  useEffect(() => {
    window.addEventListener('online', () => setIsOnline(true))
    window.addEventListener('offline', () => setIsOnline(false))
  }, [])

  return isOnline ? <AppContent /> : <OfflineBanner />
}
```

#### ❓ What is the bug in this event listener setup?

<details>
<summary>✅ Answer</summary>

```txt
The event listeners are never removed (no cleanup).

On component unmount, the listeners remain attached to the window.
If App re-mounts (e.g., in Strict Mode development, or route changes),
duplicate listeners accumulate. Each online/offline event triggers
setIsOnline multiple times unnecessarily.

In React Strict Mode (development), effects run twice — so two pairs
of listeners are registered after mount, and none are cleaned up.

Fix: Return a cleanup function:

useEffect(() => {
  const handleOnline = () => setIsOnline(true)
  const handleOffline = () => setIsOnline(false)

  window.addEventListener('online', handleOnline)
  window.addEventListener('offline', handleOffline)

  return () => {
    window.removeEventListener('online', handleOnline)
    window.removeEventListener('offline', handleOffline)
  }
}, [])
```

</details>

---

### Q24

```jsx
function useAuth() {
  const [user, setUser] = useState(null)
  const [token, setToken] = useState(localStorage.getItem('token'))

  async function fetchUserData() {
    const res = await fetch('/api/me', {
      headers: { Authorization: `Bearer ${token}` }
    })
    const data = await res.json()
    setUser(data.user)
  }

  useEffect(() => {
    if (token) fetchUserData()
  }, [token])
}
```

#### ❓ The access token expires mid-session. What does the user experience and what is the correct fix?

<details>
<summary>✅ Answer</summary>

```txt
When the token expires, /api/me returns 401.
fetch does not throw on 401 — res.json() will parse the error body.
data.user is likely undefined, so setUser(undefined) is called.
The app may redirect to login abruptly, losing the user's current work.

Broader problem: Every subsequent API call also returns 401
until the token is refreshed.

Correct fix: Implement a silent token refresh mechanism:

1. When any API call returns 401, attempt to get a new access token
   using the refresh token (stored in an HttpOnly cookie)

2. On refresh success: retry the original request with the new token
   transparently — the user sees nothing

3. On refresh failure (refresh token also expired): redirect to login

This is implemented as an Axios/fetch interceptor so ALL requests
benefit without per-call handling:

if (response.status === 401) {
  try {
    const newToken = await refreshAccessToken()
    return retry(originalRequest, newToken)
  } catch {
    clearAuthState()
    navigate('/login', { state: { reason: 'session_expired' } })
  }
}
```

</details>

---

### Q25

```jsx
function NotificationBell() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    const eventSource = new EventSource('/api/notifications/stream')
    eventSource.onmessage = (e) => {
      const data = JSON.parse(e.data)
      setCount(data.unreadCount)
    }
  }, [])

  return <Bell count={count} />
}
```

#### ❓ What happens when the user navigates away from the page and back? What is the bug?

<details>
<summary>✅ Answer</summary>

```txt
Bug 1: EventSource is never closed.
When the component unmounts (user navigates away), the SSE connection
stays open. On remount, a second connection opens. After 5 navigations,
5 connections exist simultaneously — each calling setCount on the
unmounted component (warning) and wasting server connections.

Bug 2: setCount is called on an unmounted component (memory leak warning).

Fix:

useEffect(() => {
  const eventSource = new EventSource('/api/notifications/stream')

  eventSource.onmessage = (e) => {
    const data = JSON.parse(e.data)
    setCount(data.unreadCount)
  }

  eventSource.onerror = () => {
    eventSource.close()
  }

  return () => {
    eventSource.close()    // close SSE on unmount
  }
}, [])
```

</details>

---

## ✅ Topics Covered

| Category | Count | Topics |
|---|---|---|
| Autocomplete Design | 4 | Debounce, race conditions, keyboard navigation, blur/click conflict |
| Infinite Scroll | 4 | IntersectionObserver, offset pagination, scroll jump, rootMargin |
| Real-Time | 3 | WebSocket reconnect, optimistic temp IDs, typing indicator cleanup |
| State Design | 3 | Normalization, paginated state, URL state |
| Performance | 4 | Virtualization, parallel queries, cache strategy, CLS |
| Error Handling | 3 | Retry backoff, partial failure, error boundary placement |
| Edge Cases | 3 | Double submit, offline listener cleanup, token expiry |
