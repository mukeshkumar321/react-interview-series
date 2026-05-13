# React Frontend System Design

## Table of Contents

1. [What is Frontend System Design](#1-what-is-frontend-system-design)
2. [How to Approach a Design Interview](#2-how-to-approach-a-design-interview)
3. [Design a News Feed (Facebook/Twitter)](#3-design-a-news-feed-facebooktwitter)
4. [Design an Autocomplete Component](#4-design-an-autocomplete-component)
5. [Design a Multi-Step Form](#5-design-a-multi-step-form)
6. [Design a Real-Time Chat](#6-design-a-real-time-chat)
7. [Design a Dashboard with Widgets](#7-design-a-dashboard-with-widgets)
8. [Design a File Upload Component](#8-design-a-file-upload-component)
9. [Design an Authentication Flow](#9-design-an-authentication-flow)
10. [Scalable Component Library Design](#10-scalable-component-library-design)
11. [Caching Strategies](#11-caching-strategies)
12. [Real-Time Patterns](#12-real-time-patterns)
13. [Optimistic UI](#13-optimistic-ui)
14. [Offline Support](#14-offline-support)
15. [Performance at Scale](#15-performance-at-scale)
16. [Monitoring and Observability](#16-monitoring-and-observability)
17. [Common Design Interview Questions](#17-common-design-interview-questions)

---

## 1. What is Frontend System Design

Frontend system design interviews evaluate your ability to architect scalable, maintainable, and performant client-side applications. It is fundamentally different from backend system design and requires a different mental model.

### Different from Backend System Design

| Aspect | Backend System Design | Frontend System Design |
|---|---|---|
| Primary concern | Throughput, latency, availability | User experience, perceived performance |
| Scaling unit | Servers, databases, queues | Components, bundles, rendering |
| Data storage | Databases, caches | Browser storage, memory |
| Communication | RPC, REST, gRPC | HTTP, WebSocket, SSE |
| Failure mode | Service outage | Blank screen, broken UI |
| Tooling | Load balancers, CDNs | Bundlers, service workers |

Backend system design asks: how do we handle 1 million requests per second?
Frontend system design asks: how do we render 1 million items without freezing the browser?

### Focus Areas

**Scalability** — Can the UI handle thousands of items, complex state, and many concurrent users interacting with the same shared data?

**Maintainability** — Is the code structured so that a team of 10+ engineers can work on it without constant merge conflicts? Are components composable, testable, and well-separated?

**Performance** — Does the page load under 2 seconds? Is time-to-interactive fast? Does it score well on Core Web Vitals?

**User Experience** — Are loading states handled gracefully? Are errors surfaced in a user-friendly manner? Is the UI accessible to screen reader users?

### Common Interview Topics

- Designing a UI feature from scratch (autocomplete, infinite scroll, file upload)
- Designing a full product page (news feed, dashboard, chat)
- Choosing state management strategy and data layer
- Performance optimization approaches
- Handling real-time data
- Accessibility and internationalization
- Error handling and resilience patterns

---

## 2. How to Approach a Design Interview

A structured approach demonstrates seniority. Interviewers evaluate not just what you design, but how you think through ambiguity.

### Step 1 — Clarify Requirements (2-3 minutes)

Never jump straight into design. Ask targeted questions:

**Functional requirements (what the system must do):**
- Who are the users? (unauthenticated visitors vs. logged-in users)
- What are the core user actions? (read, write, like, share)
- What is the expected scale? (100 users vs. 100 million users)
- Is there real-time data? (notifications, live updates)
- Are there mobile requirements? (responsive vs. native app)

**Non-functional requirements (quality attributes):**
- What is the target page load time?
- Is offline support needed?
- What browsers and devices must be supported?
- Are there accessibility (WCAG) requirements?
- Is internationalization (i18n) required?

### Step 2 — High-Level Architecture

Sketch the major layers of the system:

```text
User Browser
     ↓
React Application (SPA or SSR)
     ↓
API Layer (REST / GraphQL)
     ↓
Backend Services
     ↓
Database / Cache
```

For a frontend interview, you primarily own the React Application layer but must understand how it interfaces with the API layer.

### Step 3 — Component Breakdown

Decompose the UI into a component tree. Use top-down decomposition:

```text
<App>
  <Header />
  <MainContent>
    <Sidebar />
    <Feed>
      <FeedItem />
      <FeedItem />
    </Feed>
  </MainContent>
  <Footer />
</App>
```

Identify:
- **Container components** — manage state and data fetching
- **Presentational components** — receive props and render UI
- **Shared components** — used across multiple features (Button, Modal, Input)

### Step 4 — Data Model

Define the shape of data your frontend will work with:

```js
// Post data model
{
  id: string,
  author: { id: string, name: string, avatarUrl: string },
  content: string,
  imageUrl: string | null,
  likes: number,
  commentCount: number,
  createdAt: ISO8601 string,
  isLikedByCurrentUser: boolean
}
```

Normalized vs. denormalized is a key design decision. For relational data, normalization prevents duplication. For read-heavy feeds, denormalization reduces joins.

### Step 5 — State Management

Choose the right tool based on the complexity:

| Pattern | When to use |
|---|---|
| `useState` / `useReducer` | Local component state, forms |
| React Context | Global UI state (theme, auth user) |
| Redux Toolkit / Zustand | Complex cross-component state, many writers |
| React Query / SWR | Server state, caching, background refetch |
| URL state (query params) | Filter/sort/search state sharable via URL |

### Step 6 — API Design

Specify the endpoints your frontend will consume:

```txt
GET  /api/feed?cursor=<cursor>&limit=20
POST /api/posts/:id/like
GET  /api/posts/:id/comments?page=1
POST /api/posts/:id/comments
WS   /ws/feed  (real-time updates)
```

Identify:
- Pagination strategy (cursor vs. offset)
- Request deduplication
- Optimistic update contract

### Step 7 — Performance Considerations

- Code splitting: dynamic import per route
- Image optimization: lazy loading, responsive images, WebP
- Data prefetching: preload above-the-fold data
- Virtualization: for long lists (react-window or TanStack Virtual)
- Bundle size: tree shaking, no unused dependencies

### Step 8 — Edge Cases

Always call out edge cases explicitly — it signals production experience:

- Empty states (no feed items, no search results)
- Error states (API failure, network offline)
- Loading states (skeleton screens vs. spinners)
- Race conditions (debounced search, inflight requests)
- Permission errors (403, unauthorized access)
- Very long content (word break, text truncation)
- Internationalization edge cases (RTL languages, long German words)

---

## 3. Design a News Feed (Facebook/Twitter)

### Requirements

**Functional:**
- Display a chronological or ranked feed of posts
- Support infinite scroll for loading more content
- Allow liking and commenting on posts
- Show real-time updates (new posts, like counts)

**Non-functional:**
- Feed must load in under 1.5 seconds (LCP)
- Must handle 10,000+ posts without jank
- Must work on mobile devices

### Component Tree

```text
<FeedPage>
  ├── <FeedHeader />          (user info, create post button)
  ├── <CreatePost />          (textarea, media upload)
  ├── <Feed>
  │   ├── <FeedItem>          (post card)
  │   │   ├── <AuthorInfo />
  │   │   ├── <PostContent />
  │   │   ├── <PostMedia />   (lazy loaded image/video)
  │   │   ├── <LikeButton />
  │   │   └── <CommentSection>
  │   │       ├── <CommentList />
  │   │       └── <CommentInput />
  │   └── <LoadMoreTrigger /> (Intersection Observer target)
  └── <LoadingSpinner />
```

### State Design

```js
// Using React Query for server state
const {
  data,
  fetchNextPage,
  hasNextPage,
  isFetchingNextPage
} = useInfiniteQuery({
  queryKey: ['feed'],
  queryFn: ({ pageParam }) => fetchFeed(pageParam),
  getNextPageParam: (lastPage) => lastPage.nextCursor,
})

// Local UI state
const [optimisticLikes, setOptimisticLikes] = useState({})
```

### API Contract

```txt
GET  /api/feed?cursor=<cursor>&limit=20
Response:
{
  items: Post[],
  nextCursor: string | null,
  hasMore: boolean
}

POST /api/posts/:id/like
Response:
{
  postId: string,
  likes: number,
  isLiked: boolean
}

GET  /api/posts/:id/comments?page=1&limit=10
Response:
{
  comments: Comment[],
  total: number,
  page: number
}
```

### Pagination: Cursor vs. Offset

❌ Wrong — offset pagination for a live feed:
```js
// Problem: if a new post is inserted, page 2 will repeat the last item of page 1
GET /api/feed?page=2&limit=20
```

✅ Correct — cursor-based pagination:
```js
// Cursor points to the last seen item; no duplication even on inserts
GET /api/feed?cursor=eyJpZCI6IjEyMyJ9&limit=20
```

### Real-Time Updates

```jsx
useEffect(() => {
  const ws = new WebSocket('wss://api.example.com/ws/feed')

  ws.onmessage = (event) => {
    const data = JSON.parse(event.data)
    if (data.type === 'NEW_POST') {
      queryClient.setQueryData(['feed'], (old) => ({
        ...old,
        pages: [
          { items: [data.post, ...old.pages[0].items] },
          ...old.pages.slice(1)
        ]
      }))
    }
    if (data.type === 'LIKE_UPDATE') {
      updatePostLikes(data.postId, data.likes)
    }
  }

  ws.onclose = () => {
    setTimeout(() => reconnect(), 3000)
  }

  return () => ws.close()
}, [])
```

### Performance: Virtualization

```jsx
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualFeed({ posts }) {
  const parentRef = useRef()
  const rowVirtualizer = useVirtualizer({
    count: posts.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 300,
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '100vh', overflow: 'auto' }}>
      <div style={{ height: rowVirtualizer.getTotalSize() }}>
        {rowVirtualizer.getVirtualItems().map((virtualItem) => (
          <div
            key={virtualItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              transform: `translateY(${virtualItem.start}px)`,
              width: '100%',
            }}
          >
            <FeedItem post={posts[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Performance: Image Lazy Loading

```jsx
// Native lazy loading
<img src={post.imageUrl} loading="lazy" alt={post.imageAlt} />

// Intersection Observer for programmatic control
function LazyImage({ src, alt }) {
  const imgRef = useRef()
  const [loaded, setLoaded] = useState(false)

  useEffect(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) {
        setLoaded(true)
        observer.disconnect()
      }
    })
    observer.observe(imgRef.current)
    return () => observer.disconnect()
  }, [])

  return (
    <img
      ref={imgRef}
      src={loaded ? src : undefined}
      alt={alt}
    />
  )
}
```

### Caching with React Query

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60 * 1000,
      gcTime: 5 * 60 * 1000,
      refetchOnWindowFocus: true,
      retry: 2,
    }
  }
})
```

---

## 4. Design an Autocomplete Component

### Requirements

**Functional:**
- Input field with debounced search
- Dropdown list of suggestions from API
- Keyboard navigation (ArrowUp, ArrowDown, Enter, Escape)
- Select suggestion fills input
- Handle no-results state

**Non-functional:**
- Response visible under 200ms perceived latency
- Accessible to screen readers (WCAG 2.1 AA)
- Cancel in-flight requests on new keystrokes

### State Design

```js
const [query, setQuery] = useState('')
const [suggestions, setSuggestions] = useState([])
const [selectedIndex, setSelectedIndex] = useState(-1)
const [isOpen, setIsOpen] = useState(false)
const [loading, setLoading] = useState(false)
const abortControllerRef = useRef(null)
```

### Debounce Implementation

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value)

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(timer)
  }, [value, delay])

  return debouncedValue
}

function Autocomplete({ onSelect }) {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300)

  useEffect(() => {
    if (!debouncedQuery.trim()) {
      setSuggestions([])
      return
    }

    if (abortControllerRef.current) {
      abortControllerRef.current.abort()
    }

    const controller = new AbortController()
    abortControllerRef.current = controller

    setLoading(true)
    fetch(`/api/search?q=${encodeURIComponent(debouncedQuery)}`, {
      signal: controller.signal
    })
      .then(res => res.json())
      .then(data => {
        setSuggestions(data.results)
        setIsOpen(true)
      })
      .catch(err => {
        if (err.name !== 'AbortError') {
          console.error('Search failed', err)
        }
      })
      .finally(() => setLoading(false))
  }, [debouncedQuery])
}
```

### Keyboard Navigation

```jsx
function handleKeyDown(e) {
  switch (e.key) {
    case 'ArrowDown':
      e.preventDefault()
      setSelectedIndex(prev =>
        prev < suggestions.length - 1 ? prev + 1 : 0
      )
      break

    case 'ArrowUp':
      e.preventDefault()
      setSelectedIndex(prev =>
        prev > 0 ? prev - 1 : suggestions.length - 1
      )
      break

    case 'Enter':
      e.preventDefault()
      if (selectedIndex >= 0 && suggestions[selectedIndex]) {
        handleSelect(suggestions[selectedIndex])
      }
      break

    case 'Escape':
      setIsOpen(false)
      setSelectedIndex(-1)
      break

    default:
      break
  }
}
```

### Accessibility (ARIA)

```jsx
<div role="combobox" aria-expanded={isOpen} aria-haspopup="listbox">
  <input
    type="text"
    aria-autocomplete="list"
    aria-controls="suggestions-listbox"
    aria-activedescendant={
      selectedIndex >= 0 ? `suggestion-${selectedIndex}` : undefined
    }
    value={query}
    onChange={e => setQuery(e.target.value)}
    onKeyDown={handleKeyDown}
  />
  {isOpen && (
    <ul
      id="suggestions-listbox"
      role="listbox"
      aria-label="Suggestions"
    >
      {suggestions.map((item, index) => (
        <li
          key={item.id}
          id={`suggestion-${index}`}
          role="option"
          aria-selected={index === selectedIndex}
          onClick={() => handleSelect(item)}
        >
          {item.label}
        </li>
      ))}
    </ul>
  )}
</div>
```

### Race Condition Prevention

When a user types "re", "rea", "reac", "react" rapidly, four API calls fire. The responses may arrive out of order. Without cancellation, an earlier response for "re" might overwrite the response for "react".

❌ Wrong — no cancellation:
```js
useEffect(() => {
  fetch(`/api/search?q=${debouncedQuery}`)
    .then(res => res.json())
    .then(data => setSuggestions(data.results)) // stale data may win the race
}, [debouncedQuery])
```

✅ Correct — abort previous request:
```js
useEffect(() => {
  const controller = new AbortController()

  fetch(`/api/search?q=${debouncedQuery}`, { signal: controller.signal })
    .then(res => res.json())
    .then(data => setSuggestions(data.results))
    .catch(err => {
      if (err.name === 'AbortError') return
    })

  return () => controller.abort()
}, [debouncedQuery])
```

---

## 5. Design a Multi-Step Form

### Requirements

**Functional:**
- 3+ steps: Personal Info, Address, Payment, Review
- Progress indicator showing current step
- Validation per step before advancing
- Navigate forward and backward
- Auto-save draft to localStorage
- Submit on final step

### State Design

```js
const STEPS = ['Personal', 'Address', 'Payment', 'Review']

const [currentStep, setCurrentStep] = useState(0)
const [formData, setFormData] = useState({
  firstName: '',
  lastName: '',
  email: '',
  street: '',
  city: '',
  zipCode: '',
  country: '',
  cardNumber: '',
  expiryDate: '',
  cvv: '',
})
const [errors, setErrors] = useState({})
const [visitedSteps, setVisitedSteps] = useState(new Set([0]))
```

### Per-Step Validation

```js
const validationSchema = {
  0: (data) => {
    const errors = {}
    if (!data.firstName.trim()) errors.firstName = 'First name is required'
    if (!data.email.match(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)) {
      errors.email = 'Valid email is required'
    }
    return errors
  },
  1: (data) => {
    const errors = {}
    if (!data.street.trim()) errors.street = 'Street address is required'
    if (!data.zipCode.match(/^\d{5,6}$/)) errors.zipCode = 'Invalid zip code'
    return errors
  },
  2: (data) => {
    const errors = {}
    if (!data.cardNumber.match(/^\d{16}$/)) errors.cardNumber = 'Invalid card number'
    return errors
  },
}

function handleNext() {
  const stepErrors = validationSchema[currentStep]?.(formData) ?? {}
  if (Object.keys(stepErrors).length > 0) {
    setErrors(stepErrors)
    return
  }
  setErrors({})
  setCurrentStep(prev => prev + 1)
  setVisitedSteps(prev => new Set([...prev, currentStep + 1]))
}
```

### LocalStorage Draft Persistence

```jsx
useEffect(() => {
  const draft = localStorage.getItem('form-draft')
  if (draft) {
    try {
      const parsed = JSON.parse(draft)
      setFormData(parsed.formData)
      setCurrentStep(parsed.currentStep)
    } catch {
      localStorage.removeItem('form-draft')
    }
  }
}, [])

useEffect(() => {
  localStorage.setItem('form-draft', JSON.stringify({ formData, currentStep }))
}, [formData, currentStep])

async function handleSubmit() {
  const response = await submitForm(formData)
  if (response.ok) {
    localStorage.removeItem('form-draft')
    navigate('/success')
  }
}
```

### Progress Indicator Component

```jsx
function ProgressIndicator({ steps, currentStep, visitedSteps }) {
  return (
    <div
      className="progress-indicator"
      role="progressbar"
      aria-valuenow={currentStep + 1}
      aria-valuemax={steps.length}
    >
      {steps.map((step, index) => (
        <div
          key={step}
          className={[
            'step',
            index === currentStep ? 'active' : '',
            index < currentStep ? 'completed' : '',
          ].join(' ')}
          aria-current={index === currentStep ? 'step' : undefined}
        >
          <span className="step-number">{index + 1}</span>
          <span className="step-label">{step}</span>
        </div>
      ))}
    </div>
  )
}
```

---

## 6. Design a Real-Time Chat

### Requirements

**Functional:**
- Send and receive messages in real time
- Show typing indicators when other users type
- Show read receipts (delivered/read)
- Support multiple chat rooms

**Non-functional:**
- Sub-100ms message latency
- Graceful reconnection on disconnect
- Optimistic message rendering

### WebSocket Architecture

```text
Browser <────────── WebSocket ──────────> Chat Server
   ↓                                           ↓
React State                          Message Broker (Redis Pub/Sub)
   ↓                                           ↓
UI Render                             Database (PostgreSQL)
```

### State Design

```js
const [messages, setMessages] = useState([])
const [typingUsers, setTypingUsers] = useState([])
const [connected, setConnected] = useState(false)
const [messageInput, setMessageInput] = useState('')
const wsRef = useRef(null)
const typingTimeoutRef = useRef({})
```

### WebSocket Connection Hook

```jsx
function useChatSocket(roomId, userId) {
  const [messages, setMessages] = useState([])
  const [typingUsers, setTypingUsers] = useState([])
  const [connected, setConnected] = useState(false)
  const wsRef = useRef(null)

  const connect = useCallback(() => {
    const ws = new WebSocket(`wss://chat.example.com/rooms/${roomId}`)

    ws.onopen = () => setConnected(true)

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data)
      switch (msg.type) {
        case 'MESSAGE':
          setMessages(prev => [...prev, msg.data])
          break
        case 'TYPING_START':
          setTypingUsers(prev => [...new Set([...prev, msg.userId])])
          break
        case 'TYPING_STOP':
          setTypingUsers(prev => prev.filter(id => id !== msg.userId))
          break
        case 'READ_RECEIPT':
          setMessages(prev => prev.map(m =>
            m.id === msg.messageId ? { ...m, readBy: msg.userId } : m
          ))
          break
      }
    }

    ws.onclose = () => {
      setConnected(false)
      setTimeout(connect, 3000)
    }

    ws.onerror = () => ws.close()
    wsRef.current = ws
  }, [roomId])

  useEffect(() => {
    connect()
    return () => wsRef.current?.close()
  }, [connect])

  const sendMessage = useCallback((text) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify({
        type: 'MESSAGE',
        roomId,
        text,
        tempId: crypto.randomUUID(),
      }))
    }
  }, [roomId])

  return { messages, typingUsers, connected, sendMessage }
}
```

### Optimistic Message Sending

```jsx
function sendMessage(text) {
  const tempId = crypto.randomUUID()
  const optimisticMessage = {
    id: tempId,
    text,
    senderId: currentUserId,
    timestamp: new Date().toISOString(),
    status: 'sending',
  }

  setMessages(prev => [...prev, optimisticMessage])
  ws.send(JSON.stringify({ type: 'MESSAGE', text, tempId }))
}

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data)
  if (msg.type === 'MESSAGE_CONFIRMED') {
    setMessages(prev => prev.map(m =>
      m.id === msg.tempId
        ? { ...msg.data, status: 'sent' }
        : m
    ))
  }
  if (msg.type === 'MESSAGE_FAILED') {
    setMessages(prev => prev.map(m =>
      m.id === msg.tempId
        ? { ...m, status: 'failed' }
        : m
    ))
  }
}
```

### Typing Indicator with Cleanup

```jsx
const TYPING_TIMEOUT = 2000

function handleInputChange(e) {
  setMessageInput(e.target.value)

  ws.send(JSON.stringify({ type: 'TYPING_START', roomId, userId: currentUserId }))

  clearTimeout(typingTimeoutRef.current)
  typingTimeoutRef.current = setTimeout(() => {
    ws.send(JSON.stringify({ type: 'TYPING_STOP', roomId, userId: currentUserId }))
  }, TYPING_TIMEOUT)
}

useEffect(() => {
  return () => {
    clearTimeout(typingTimeoutRef.current)
    ws.send(JSON.stringify({ type: 'TYPING_STOP', roomId, userId: currentUserId }))
  }
}, [])
```

---

## 7. Design a Dashboard with Widgets

### Widget Architecture

A dashboard is a configurable grid of independent widgets. Each widget owns its data fetching, loading state, and error handling independently.

```jsx
const WIDGET_REGISTRY = {
  sales_chart: SalesChartWidget,
  user_stats: UserStatsWidget,
  recent_orders: RecentOrdersWidget,
  revenue_kpi: RevenueKPIWidget,
}

const dashboardConfig = {
  layout: [
    { id: 'w1', type: 'sales_chart', x: 0, y: 0, w: 6, h: 2 },
    { id: 'w2', type: 'user_stats', x: 6, y: 0, w: 6, h: 2 },
    { id: 'w3', type: 'recent_orders', x: 0, y: 2, w: 12, h: 3 },
  ]
}
```

### Widget Wrapper with Error Boundary and Permissions

```jsx
function WidgetWrapper({ config, permissions }) {
  const WidgetComponent = WIDGET_REGISTRY[config.type]

  if (!permissions.includes(config.type)) {
    return <PermissionDeniedWidget name={config.type} />
  }

  if (!WidgetComponent) {
    return <UnknownWidget type={config.type} />
  }

  return (
    <ErrorBoundary fallback={<WidgetErrorState />}>
      <Suspense fallback={<WidgetSkeleton />}>
        <WidgetComponent
          id={config.id}
          refreshInterval={config.refreshInterval}
        />
      </Suspense>
    </ErrorBoundary>
  )
}
```

### Per-Widget Auto Refresh

```jsx
function SalesChartWidget({ id, refreshInterval = 60000 }) {
  const { data, error, isLoading, refetch } = useQuery({
    queryKey: ['widget', id, 'sales'],
    queryFn: () => fetchWidgetData(id),
    refetchInterval: refreshInterval,
    refetchIntervalInBackground: false,
  })

  if (isLoading) return <WidgetSkeleton />
  if (error) return <WidgetErrorState onRetry={refetch} />

  return <SalesChart data={data} />
}
```

### Configurable Layout (react-grid-layout)

```jsx
import GridLayout from 'react-grid-layout'

function Dashboard({ config, onLayoutChange }) {
  return (
    <GridLayout
      layout={config.layout}
      cols={12}
      rowHeight={150}
      onLayoutChange={onLayoutChange}
      draggableHandle=".widget-drag-handle"
    >
      {config.layout.map(widget => (
        <div key={widget.id}>
          <WidgetWrapper config={widget} permissions={userPermissions} />
        </div>
      ))}
    </GridLayout>
  )
}
```

---

## 8. Design a File Upload Component

### Requirements

**Functional:**
- Drag and drop interface
- Multiple file selection
- Per-file upload progress bar
- Cancel individual upload
- Retry failed uploads
- Chunked upload for large files (> 10 MB)

### State Design

```js
// File entry shape
{
  id: string,
  file: File,
  status: 'pending' | 'uploading' | 'success' | 'error' | 'cancelled',
  progress: number,    // 0-100
  error: string | null,
  abortController: AbortController,
  uploadedUrl: string | null,
}

const [files, setFiles] = useState([])
```

### Drag and Drop Implementation

```jsx
function FileDropZone({ onFilesAdded }) {
  const [isDragging, setIsDragging] = useState(false)
  const dropRef = useRef()

  const handleDrop = useCallback((e) => {
    e.preventDefault()
    setIsDragging(false)
    const files = Array.from(e.dataTransfer.files)
    onFilesAdded(files)
  }, [onFilesAdded])

  const handleDragOver = useCallback((e) => {
    e.preventDefault()
    setIsDragging(true)
  }, [])

  const handleDragLeave = useCallback((e) => {
    if (!dropRef.current.contains(e.relatedTarget)) {
      setIsDragging(false)
    }
  }, [])

  return (
    <div
      ref={dropRef}
      onDrop={handleDrop}
      onDragOver={handleDragOver}
      onDragLeave={handleDragLeave}
      className={`drop-zone ${isDragging ? 'dragging' : ''}`}
      role="button"
      aria-label="Drop files here or click to select"
    >
      Drop files here or{' '}
      <label>
        click to browse
        <input
          type="file"
          multiple
          hidden
          onChange={e => {
            onFilesAdded(Array.from(e.target.files))
            e.target.value = ''
          }}
        />
      </label>
    </div>
  )
}
```

### XMLHttpRequest with Progress and Cancel

```js
function uploadFile(fileEntry, onProgress, onComplete, onError) {
  const { file, abortController } = fileEntry
  const formData = new FormData()
  formData.append('file', file)

  const xhr = new XMLHttpRequest()

  xhr.upload.onprogress = (event) => {
    if (event.lengthComputable) {
      const percent = Math.round((event.loaded / event.total) * 100)
      onProgress(fileEntry.id, percent)
    }
  }

  xhr.onload = () => {
    if (xhr.status >= 200 && xhr.status < 300) {
      const response = JSON.parse(xhr.responseText)
      onComplete(fileEntry.id, response.url)
    } else {
      onError(fileEntry.id, `Upload failed: ${xhr.statusText}`)
    }
  }

  xhr.onerror = () => onError(fileEntry.id, 'Network error')
  abortController.signal.addEventListener('abort', () => xhr.abort())

  xhr.open('POST', '/api/upload')
  xhr.send(formData)
}
```

### Chunked Upload for Large Files

```js
const CHUNK_SIZE = 5 * 1024 * 1024 // 5 MB per chunk

async function uploadInChunks(file, uploadId, onProgress) {
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE)

  for (let chunkIndex = 0; chunkIndex < totalChunks; chunkIndex++) {
    const start = chunkIndex * CHUNK_SIZE
    const end = Math.min(start + CHUNK_SIZE, file.size)
    const chunk = file.slice(start, end)

    const formData = new FormData()
    formData.append('chunk', chunk)
    formData.append('uploadId', uploadId)
    formData.append('chunkIndex', chunkIndex)
    formData.append('totalChunks', totalChunks)

    await fetch('/api/upload/chunk', { method: 'POST', body: formData })

    const progress = Math.round(((chunkIndex + 1) / totalChunks) * 100)
    onProgress(progress)
  }

  // Instruct server to assemble chunks
  await fetch('/api/upload/finalize', {
    method: 'POST',
    body: JSON.stringify({ uploadId }),
    headers: { 'Content-Type': 'application/json' }
  })
}
```

---

## 9. Design an Authentication Flow

### Token Storage Strategy

| Storage | XSS Safe | CSRF Safe | Notes |
|---|---|---|---|
| `localStorage` | No | Yes | Accessible via JS; XSS exposes token |
| `sessionStorage` | No | Yes | Cleared on tab close |
| HttpOnly Cookie | Yes | No | Requires CSRF token header |
| Memory (React state) | Yes | Yes | Lost on page refresh |

**Recommended pattern:** Access token in memory + Refresh token in HttpOnly cookie. The refresh token cannot be read by JavaScript. A CSRF token sent as a header prevents cross-site request forgery on the refresh endpoint.

### Protected Route Implementation

```jsx
function ProtectedRoute({ children }) {
  const { user, isLoading } = useAuth()
  const location = useLocation()

  if (isLoading) return <PageLoader />

  if (!user) {
    return <Navigate to="/login" state={{ from: location.pathname }} replace />
  }

  return children
}

function LoginPage() {
  const { state } = useLocation()
  const navigate = useNavigate()

  async function handleLogin(credentials) {
    await login(credentials)
    navigate(state?.from ?? '/dashboard', { replace: true })
  }
}
```

### Token Refresh Interceptor (Axios)

```js
let isRefreshing = false
let failedQueue = []

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error)
    else prom.resolve(token)
  })
  failedQueue = []
}

axiosInstance.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject })
        }).then(token => {
          originalRequest.headers['Authorization'] = `Bearer ${token}`
          return axiosInstance(originalRequest)
        })
      }

      originalRequest._retry = true
      isRefreshing = true

      try {
        const { accessToken } = await refreshTokenRequest()
        setAccessToken(accessToken)
        processQueue(null, accessToken)
        originalRequest.headers['Authorization'] = `Bearer ${accessToken}`
        return axiosInstance(originalRequest)
      } catch (refreshError) {
        processQueue(refreshError, null)
        logout()
        return Promise.reject(refreshError)
      } finally {
        isRefreshing = false
      }
    }

    return Promise.reject(error)
  }
)
```

### Auth Context Structure

```jsx
const AuthContext = createContext(null)

function AuthProvider({ children }) {
  const [user, setUser] = useState(null)
  const [isLoading, setIsLoading] = useState(true)
  const accessTokenRef = useRef(null)

  useEffect(() => {
    // On app load, attempt silent refresh using the HttpOnly cookie
    refreshTokenRequest()
      .then(({ accessToken, user }) => {
        accessTokenRef.current = accessToken
        setUser(user)
      })
      .catch(() => setUser(null))
      .finally(() => setIsLoading(false))
  }, [])

  const login = async (credentials) => {
    const { accessToken, user } = await loginRequest(credentials)
    accessTokenRef.current = accessToken
    setUser(user)
  }

  const logout = async () => {
    await logoutRequest()
    accessTokenRef.current = null
    setUser(null)
  }

  return (
    <AuthContext.Provider value={{ user, isLoading, login, logout }}>
      {children}
    </AuthContext.Provider>
  )
}
```

---

## 10. Scalable Component Library Design

### Public API Design Principles

A component's public API is its `props` interface. Design it to be minimal, composable, and forward-compatible.

```jsx
// Well-designed Button component
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost'
  size?: 'sm' | 'md' | 'lg'
  loading?: boolean
  leftIcon?: React.ReactNode
  rightIcon?: React.ReactNode
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', size = 'md', loading, leftIcon, rightIcon,
     children, disabled, ...rest }, ref) => {
    return (
      <button
        ref={ref}
        disabled={disabled || loading}
        aria-busy={loading}
        className={clsx('btn', `btn--${variant}`, `btn--${size}`, {
          'btn--loading': loading
        })}
        {...rest}
      >
        {loading && <Spinner size="sm" />}
        {!loading && leftIcon}
        {children}
        {!loading && rightIcon}
      </button>
    )
  }
)
```

### Theming with CSS Custom Properties

```css
:root {
  --color-primary: #0070f3;
  --color-primary-hover: #0051cc;
  --color-danger: #e00;
  --color-surface: #ffffff;
  --radius-sm: 4px;
  --radius-md: 8px;
  --spacing-1: 4px;
  --spacing-2: 8px;
  --font-size-sm: 14px;
  --font-size-md: 16px;
}

[data-theme="dark"] {
  --color-primary: #3b9eff;
  --color-surface: #1a1a1a;
}
```

```jsx
function ThemeProvider({ theme, children }) {
  useLayoutEffect(() => {
    const root = document.documentElement
    Object.entries(theme.tokens).forEach(([key, value]) => {
      root.style.setProperty(`--${key}`, value)
    })
    root.setAttribute('data-theme', theme.name)
  }, [theme])

  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  )
}
```

### Compound Component Pattern

```jsx
// Compound pattern allows flexible composition
function Tabs({ children, defaultIndex = 0 }) {
  const [activeIndex, setActiveIndex] = useState(defaultIndex)
  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      {children}
    </TabsContext.Provider>
  )
}

Tabs.List = function TabList({ children }) {
  return <div role="tablist">{children}</div>
}

Tabs.Tab = function Tab({ children, index }) {
  const { activeIndex, setActiveIndex } = useContext(TabsContext)
  return (
    <button
      role="tab"
      aria-selected={activeIndex === index}
      onClick={() => setActiveIndex(index)}
    >
      {children}
    </button>
  )
}

Tabs.Panel = function TabPanel({ children, index }) {
  const { activeIndex } = useContext(TabsContext)
  return activeIndex === index ? <div role="tabpanel">{children}</div> : null
}

// Usage
<Tabs defaultIndex={0}>
  <Tabs.List>
    <Tabs.Tab index={0}>Overview</Tabs.Tab>
    <Tabs.Tab index={1}>Details</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel index={0}><OverviewContent /></Tabs.Panel>
  <Tabs.Panel index={1}><DetailsContent /></Tabs.Panel>
</Tabs>
```

### Versioning Strategy

```txt
MAJOR (2.0.0) — breaking changes
  - Removed prop or changed its type
  - Changed DOM structure in a way that breaks CSS selectors
  - Removed an exported component or hook

MINOR (1.1.0) — backwards-compatible additions
  - New optional prop
  - New exported component
  - New variant value

PATCH (1.0.1) — bug fixes only
  - Fixed focus ring visibility
  - Fixed ARIA attribute typo
```

---

## 11. Caching Strategies

### Browser Cache (HTTP Headers)

```txt
Static assets with content hash (bundle.abc123.js):
  Cache-Control: max-age=31536000, immutable
  → Browser never re-requests for 1 year

index.html (entry point, must not be stale):
  Cache-Control: no-cache
  ETag: "abc123"
  → Browser validates with ETag; server returns 304 if unchanged

API responses (variable freshness):
  Cache-Control: private, max-age=60
  → Cached in browser only (not CDN) for 60 seconds
```

### React Query Cache Layers

```text
Request fired
     ↓
Is query in cache AND stale=false?
     ↓ YES → return cached data immediately (no network request)
     ↓ NO
Is query in cache AND stale=true?
     ↓ YES → return stale data immediately, fire background refetch
     ↓ NO
No cache → show loading state, fire network request
```

```jsx
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 5 * 60 * 1000,    // data is fresh for 5 minutes
  gcTime: 30 * 60 * 1000,       // removed from memory after 30 min inactive
  refetchOnWindowFocus: true,
  refetchOnReconnect: true,
  placeholderData: keepPreviousData, // show old data while fetching page 2
})
```

### Cache Invalidation

```jsx
const { mutate: createPost } = useMutation({
  mutationFn: (post) => api.createPost(post),
  onSuccess: () => {
    // Invalidate and refetch
    queryClient.invalidateQueries({ queryKey: ['posts'] })
  },
  onMutate: async (newPost) => {
    await queryClient.cancelQueries({ queryKey: ['posts'] })
    const previous = queryClient.getQueryData(['posts'])
    queryClient.setQueryData(['posts'], old => [newPost, ...old])
    return { previous }
  },
  onError: (err, newPost, context) => {
    queryClient.setQueryData(['posts'], context.previous)
  },
})
```

---

## 12. Real-Time Patterns

### WebSockets

Full-duplex communication over a persistent TCP connection. Optimal for chat, collaborative editing, multiplayer games.

```js
const ws = new WebSocket('wss://api.example.com/ws')
ws.onopen = () => ws.send(JSON.stringify({ type: 'SUBSCRIBE', channel: 'feed' }))
ws.onmessage = (e) => dispatch(handleServerMessage(JSON.parse(e.data)))
ws.onclose = () => scheduleReconnect()
```

### Server-Sent Events (SSE)

One-way push from server to client over standard HTTP. Automatically reconnects. Best for notifications, live feeds, progress updates.

```js
const eventSource = new EventSource('/api/events')
eventSource.onmessage = (e) => handleUpdate(JSON.parse(e.data))
eventSource.addEventListener('ORDER_UPDATE', (e) => handleOrder(JSON.parse(e.data)))
// Auto-reconnects with Last-Event-ID header on disconnect
```

### Long Polling

Client holds an open request; server responds when data is available.

```js
async function longPoll(lastEventId) {
  try {
    const res = await fetch(`/api/poll?lastId=${lastEventId}`, {
      headers: { 'Cache-Control': 'no-cache' }
    })
    const data = await res.json()
    handleEvents(data.events)
    longPoll(data.lastId)
  } catch {
    setTimeout(() => longPoll(lastEventId), 5000)
  }
}
```

### Comparison Table

| Feature | WebSocket | SSE | Long Polling |
|---|---|---|---|
| Direction | Bi-directional | Server to client | Server to client |
| Protocol | WS / WSS | HTTP | HTTP |
| Auto-reconnect | No (manual) | Yes | Manual |
| Firewall friendly | Sometimes | Yes | Yes |
| Binary support | Yes | No (text only) | No |
| Overhead | Low | Low | High (HTTP headers each poll) |
| Best for | Chat, gaming, collab | Feeds, notifications | Legacy browser fallback |

---

## 13. Optimistic UI

### When to Apply

✅ Good candidates:
- Liking a post
- Toggling a bookmark
- Marking a notification as read
- Adding an item to a shopping cart

❌ Not appropriate for:
- Payment transactions (must confirm server response)
- Deleting irreversible data without undo
- Operations with multi-step server-side effects

### Full Implementation with React Query

```jsx
const { mutate: toggleLike } = useMutation({
  mutationFn: (postId) => api.toggleLike(postId),

  onMutate: async (postId) => {
    // Cancel any in-flight refetches that would overwrite the optimistic update
    await queryClient.cancelQueries({ queryKey: ['post', postId] })

    // Snapshot the current value
    const previousPost = queryClient.getQueryData(['post', postId])

    // Optimistically update to the new value
    queryClient.setQueryData(['post', postId], (old) => ({
      ...old,
      isLiked: !old.isLiked,
      likes: old.isLiked ? old.likes - 1 : old.likes + 1
    }))

    return { previousPost }
  },

  onError: (err, postId, context) => {
    // Roll back to the snapshot on error
    queryClient.setQueryData(['post', postId], context.previousPost)
    showToast('Could not update like. Please try again.')
  },

  onSettled: (postId) => {
    // Always refetch after error or success to sync with server truth
    queryClient.invalidateQueries({ queryKey: ['post', postId] })
  },
})
```

---

## 14. Offline Support

### Service Worker Strategies

```text
Cache-First (static assets):
  Request → Cache → return if hit → else Network → cache and return

Network-First (API calls):
  Request → Network → return and update cache → on failure → Cache fallback

Stale-While-Revalidate (semi-dynamic):
  Request → Cache (return immediately) + Network (update cache for next time)
```

### Service Worker Implementation

```js
// service-worker.js
const CACHE_NAME = 'app-v1'
const STATIC_ASSETS = ['/', '/index.html', '/static/bundle.js', '/static/styles.css']

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(STATIC_ASSETS))
  )
  self.skipWaiting()
})

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  )
  self.clients.claim()
})

self.addEventListener('fetch', (event) => {
  const { request } = event
  const url = new URL(request.url)

  if (url.pathname.startsWith('/api/')) {
    // Network-first for API
    event.respondWith(
      fetch(request)
        .then(response => {
          const clone = response.clone()
          caches.open(CACHE_NAME).then(cache => cache.put(request, clone))
          return response
        })
        .catch(() => caches.match(request))
    )
  } else {
    // Cache-first for static
    event.respondWith(
      caches.match(request).then(cached => cached ?? fetch(request))
    )
  }
})
```

### IndexedDB for Offline Write Queue

```js
import { openDB } from 'idb'

const db = await openDB('appDB', 1, {
  upgrade(db) {
    db.createObjectStore('outbox', { keyPath: 'id', autoIncrement: true })
  }
})

async function sendOrQueue(action) {
  if (navigator.onLine) {
    await api.perform(action)
  } else {
    await db.add('outbox', { action, timestamp: Date.now() })
  }
}

window.addEventListener('online', async () => {
  const pending = await db.getAll('outbox')
  for (const item of pending) {
    try {
      await api.perform(item.action)
      await db.delete('outbox', item.id)
    } catch {
      break // stop on first failure; retry next time
    }
  }
})
```

---

## 15. Performance at Scale

### Core Web Vitals Targets

| Metric | Good | Needs Improvement | Poor |
|---|---|---|---|
| LCP (Largest Contentful Paint) | < 2.5s | 2.5s – 4s | > 4s |
| FID (First Input Delay) | < 100ms | 100ms – 300ms | > 300ms |
| CLS (Cumulative Layout Shift) | < 0.1 | 0.1 – 0.25 | > 0.25 |
| INP (Interaction to Next Paint) | < 200ms | 200ms – 500ms | > 500ms |

### Code Splitting

```jsx
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Profile = lazy(() => import('./pages/Profile'))
const AdminPanel = lazy(() => import('./pages/AdminPanel'))

function App() {
  return (
    <Suspense fallback={<PageSpinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
        <Route path="/admin" element={
          <ProtectedRoute requiredRole="admin">
            <AdminPanel />
          </ProtectedRoute>
        } />
      </Routes>
    </Suspense>
  )
}
```

### Bundle Size Reduction

```bash
# Analyze what is in the bundle
npx webpack-bundle-analyzer dist/stats.json

# Common wins:
# Replace moment.js (72kB gzipped) with date-fns (tree-shakeable, ~3kB per function)
# Use lodash-es instead of lodash (enables tree shaking)
# Use react-icons/{fa} not react-icons (imports only what you use)
# Dynamic import for heavy modals/editors not needed on initial load
```

### Preloading Critical Resources

```html
<!-- Preload the LCP image (hero image above fold) -->
<link rel="preload" href="/hero.webp" as="image" fetchpriority="high">

<!-- Preconnect to API domain to shorten TCP handshake -->
<link rel="preconnect" href="https://api.example.com">

<!-- Prefetch next-page JS bundle when user hovers a nav link -->
<link rel="prefetch" href="/static/Settings.chunk.js">
```

### React Performance Patterns

```jsx
// memo prevents re-render when props are identical
const FeedItem = memo(function FeedItem({ post, onLike }) {
  return <div>{post.title}</div>
})

// useCallback stabilizes function references passed as props
const handleLike = useCallback((postId) => {
  mutate(postId)
}, [mutate])

// useMemo memoizes expensive computations
const sortedPosts = useMemo(() =>
  [...posts].sort((a, b) => b.timestamp - a.timestamp),
  [posts]
)

// startTransition defers non-urgent state updates
import { startTransition } from 'react'

function handleSearch(query) {
  setInputValue(query)              // urgent — update input immediately
  startTransition(() => {
    setSearchQuery(query)           // deferred — search results can wait
  })
}
```

---

## 16. Monitoring and Observability

### Error Tracking with Sentry

```jsx
import * as Sentry from '@sentry/react'

Sentry.init({
  dsn: process.env.REACT_APP_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.2,
  replaysOnErrorSampleRate: 1.0,
  integrations: [
    Sentry.reactRouterV6BrowserTracingIntegration({ useEffect }),
    Sentry.replayIntegration(),
  ],
})

// Wrap root with Sentry error boundary
const App = Sentry.withErrorBoundary(RootApp, {
  fallback: <ErrorFallback />,
})

// Manual capture with context
async function handlePayment(data) {
  try {
    await processPayment(data)
  } catch (err) {
    Sentry.captureException(err, {
      user: { id: user.id, email: user.email },
      tags: { feature: 'checkout' },
      extra: { orderId: data.orderId },
    })
    throw err
  }
}
```

### Web Vitals Reporting

```js
import { onCLS, onFID, onLCP, onINP, onTTFB } from 'web-vitals'

function reportMetric(metric) {
  analytics.track('web_vital', {
    name: metric.name,
    value: Math.round(metric.value),
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    page: window.location.pathname,
    connectionType: navigator.connection?.effectiveType,
  })
}

onCLS(reportMetric)
onFID(reportMetric)
onLCP(reportMetric)
onINP(reportMetric)
onTTFB(reportMetric)
```

### Feature Flags for Safe Rollouts

```jsx
// Gradual rollout: 10% of users see new feature
function useFeatureFlag(flagName) {
  const { user } = useAuth()
  return launchDarkly.variation(flagName, {
    key: user.id,
    email: user.email,
    plan: user.plan,
  }, false)
}

function FeedPage() {
  const useNewAlgorithm = useFeatureFlag('feed-ranking-v2')
  return useNewAlgorithm ? <NewFeed /> : <OldFeed />
}
```

---

## 17. Common Design Interview Questions

### Design Google Search Autocomplete

**Key decisions to mention:**
- Debounce 200–300ms on keystrokes to limit API calls
- Cache previous results by query string — if user re-types "re" after deleting "react", return cached results without a new request
- Cancel in-flight request with AbortController before each new fetch
- Keyboard navigation: ArrowUp/Down selects suggestion, Enter submits, Escape closes dropdown
- ARIA: role="combobox", aria-expanded, aria-activedescendant
- Session tokens: Google Autocomplete API groups keystrokes into a session for billing

### Design a Calendar Widget

**Key decisions to mention:**
- State: currentMonth (Date), selectedDate (Date | null), events (Event[])
- Views: month / week / day — each is a separate component, controlled by a view selector
- Event overlap calculation: sort by start time, assign column indices for overlapping events
- Recurring events should be expanded on the server to a date range, not computed on client
- Timezone: store all times as UTC, display using Intl.DateTimeFormat with user's timezone
- Accessibility: keyboard navigation between dates (arrow keys), role="grid", role="gridcell"

### Design an Infinite Scroll List

**Key decisions to mention:**
- IntersectionObserver on a sentinel element at the bottom of the list (not scroll event listeners)
- Cursor-based pagination — no item duplication when new items are inserted at the top
- Virtualize the list when items can exceed 200 rows (TanStack Virtual, react-window)
- Skeleton loaders for next-page load, not a full-page spinner
- hasMore flag to prevent pointless API calls after last page is reached
- Handle browser back button: restore scroll position using scrollRestoration

### Design a Notification System

**Key decisions to mention:**
- Real-time delivery: SSE for simplicity (one-way push), WebSocket if bidirectional acknowledgment needed
- Unread count: maintained in auth context, decremented optimistically on read
- Mark as read: optimistic update, rollback if API fails
- Notification grouping: "Alice and 5 others liked your post" — group by entity + action on server
- Persistence: store unread count in localStorage so badge survives page refresh before API loads

### Design a Shopping Cart

**Key decisions to mention:**
- State shape: `{ items: { [productId]: { product, quantity } } }` — normalized by productId prevents duplicates
- Guest cart: persist in localStorage; on login, merge with server-saved cart (server takes precedence)
- Quantity update: optimistic, debounce the API call by 500ms to avoid a call per click
- Stock validation: validate at checkout (not on add-to-cart) to keep UX fast
- Cart count badge: derive from state, do not store separately

---
