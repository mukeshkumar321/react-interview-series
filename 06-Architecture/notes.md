# React Architecture

## Table of Contents

- [1. Component Design Principles](#1-component-design-principles)
- [2. Component Patterns](#2-component-patterns)
- [3. Project and Folder Structure](#3-project-and-folder-structure)
- [4. State Management Strategy](#4-state-management-strategy)
- [5. Data Flow Patterns](#5-data-flow-patterns)
- [6. Code Splitting Strategies](#6-code-splitting-strategies)
- [7. Micro-Frontend Concepts](#7-micro-frontend-concepts)
- [8. Server Components vs Client Components](#8-server-components-vs-client-components)
- [9. Rendering Patterns](#9-rendering-patterns)
- [10. Error Boundaries](#10-error-boundaries)
- [11. Suspense and Concurrent Features](#11-suspense-and-concurrent-features)
- [12. Performance Patterns](#12-performance-patterns)
- [13. Testing Strategy](#13-testing-strategy)
- [14. Common Architectural Mistakes](#14-common-architectural-mistakes)
- [15. Best Practices](#15-best-practices)

---

## 1. Component Design Principles

### Single Responsibility Principle (SRP)

Each component should have one clear reason to change. A component that fetches data, formats it, validates it, and renders it violates SRP.

```jsx
// ❌ Wrong — one component does everything
function UserDashboard({ userId }) {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);

  useEffect(() => { fetchUser(userId).then(setUser); }, [userId]);
  useEffect(() => { fetchPosts(userId).then(setPosts); }, [userId]);

  const formattedDate = new Date(user?.createdAt).toLocaleDateString();

  const handlePostDelete = (id) => {
    deletePost(id).then(() => setPosts(p => p.filter(x => x.id !== id)));
  };

  return (
    <div>
      <img src={user?.avatar} />
      <h1>{user?.name}</h1>
      <p>Member since: {formattedDate}</p>
      {posts.map(post => (
        <div key={post.id}>
          <p>{post.title}</p>
          <button onClick={() => handlePostDelete(post.id)}>Delete</button>
        </div>
      ))}
    </div>
  );
}

// ✅ Correct — split by responsibility
function UserDashboard({ userId }) {
  return (
    <div>
      <UserProfile userId={userId} />
      <UserPosts userId={userId} />
    </div>
  );
}
```

### Composition Over Inheritance

React does not use class inheritance for component reuse. Composition — passing children and rendering them — is the idiomatic way to share UI structure.

```jsx
// ❌ Wrong — inheritance approach
class SpecialButton extends Button {
  render() { ... }
}

// ✅ Correct — composition
function Card({ header, children, footer }) {
  return (
    <div className="card">
      <div className="card-header">{header}</div>
      <div className="card-body">{children}</div>
      <div className="card-footer">{footer}</div>
    </div>
  );
}

// Usage — flexible, composable
<Card
  header={<UserAvatar user={user} />}
  footer={<PostActions post={post} />}
>
  <PostContent post={post} />
</Card>
```

### Separation of Concerns

Separate UI rendering, data fetching, business logic, and state management into distinct layers.

```text
Component Layer
  ↓ reads from
Custom Hook Layer  (useUserData, usePostActions)
  ↓ calls
Service Layer      (userService.ts, postService.ts)
  ↓ communicates with
API Layer          (/api/users, /api/posts)
```

---

## 2. Component Patterns

### Presentational vs Container Components

| | Presentational | Container |
|---|---|---|
| Concern | How things look | How things work |
| Data source | Props | State, hooks, context |
| Side effects | None | Yes (fetching, subscriptions) |
| Reusability | High | Lower |
| Testing | Easy (pure render) | Requires mocking |

```jsx
// Container — concerned with data
function UserCardContainer({ userId }) {
  const { user, isLoading, error } = useUser(userId);

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;
  return <UserCard user={user} />;
}

// Presentational — concerned only with display
function UserCard({ user }) {
  return (
    <div className="user-card">
      <img src={user.avatar} alt={user.name} />
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

### Compound Components Pattern

Compound components share implicit state through context, allowing flexible composition without passing many props.

```jsx
// Compound component implementation
const TabContext = createContext();

function Tabs({ children, defaultTab }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabContext.Provider>
  );
}

Tabs.List = function TabList({ children }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Tab = function Tab({ id, children }) {
  const { activeTab, setActiveTab } = useContext(TabContext);
  return (
    <button
      role="tab"
      aria-selected={activeTab === id}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  );
};

Tabs.Panel = function TabPanel({ id, children }) {
  const { activeTab } = useContext(TabContext);
  if (activeTab !== id) return null;
  return <div role="tabpanel">{children}</div>;
};

// Usage — flexible, no prop drilling
<Tabs defaultTab="profile">
  <Tabs.List>
    <Tabs.Tab id="profile">Profile</Tabs.Tab>
    <Tabs.Tab id="settings">Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel id="profile"><ProfileForm /></Tabs.Panel>
  <Tabs.Panel id="settings"><SettingsForm /></Tabs.Panel>
</Tabs>
```

### Render Props Pattern

A component exposes a function prop (usually called `render` or `children`) that it calls with internal data. The caller controls what is rendered.

```jsx
// Render props implementation
function MouseTracker({ children }) {
  const [position, setPosition] = useState({ x: 0, y: 0 });

  const handleMouseMove = (e) => {
    setPosition({ x: e.clientX, y: e.clientY });
  };

  return (
    <div onMouseMove={handleMouseMove}>
      {children(position)}
    </div>
  );
}

// Usage — caller decides what to do with position
<MouseTracker>
  {({ x, y }) => (
    <p>Mouse at {x}, {y}</p>
  )}
</MouseTracker>

<MouseTracker>
  {({ x, y }) => (
    <Crosshair top={y} left={x} />
  )}
</MouseTracker>
```

Render props are largely superseded by custom hooks, but you will still encounter them in older codebases and libraries.

### Higher-Order Components (HOC)

A HOC is a function that takes a component and returns a new, enhanced component. It is a wrapper pattern.

```jsx
// HOC for authentication check
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const { user, isLoading } = useAuth();

    if (isLoading) return <Spinner />;
    if (!user) return <Navigate to="/login" replace />;

    return <WrappedComponent {...props} user={user} />;
  };
}

// Usage
const ProtectedProfile = withAuth(Profile);
const ProtectedDashboard = withAuth(Dashboard);

// HOC for analytics tracking
function withAnalytics(WrappedComponent, eventName) {
  return function TrackedComponent(props) {
    useEffect(() => {
      analytics.track(`view_${eventName}`);
    }, []);

    return <WrappedComponent {...props} />;
  };
}
```

HOC Caveats:
- Wrapping creates an extra layer in DevTools (use `displayName` to fix this)
- Passing refs requires `React.forwardRef`
- Props conflict can occur if the HOC and wrapped component use the same prop name
- Custom hooks are often a cleaner alternative

```jsx
// Set displayName for better DevTools experience
function withAuth(WrappedComponent) {
  function AuthenticatedComponent(props) { ... }
  AuthenticatedComponent.displayName = `withAuth(${
    WrappedComponent.displayName || WrappedComponent.name
  })`;
  return AuthenticatedComponent;
}
```

---

## 3. Project and Folder Structure

### Type-Based Structure

Organizes files by technical concern. Common in small projects.

```text
src/
  components/
    Button.jsx
    Modal.jsx
    UserCard.jsx
  hooks/
    useAuth.js
    useUser.js
  pages/
    Home.jsx
    Dashboard.jsx
  services/
    api.js
    userService.js
  store/
    userSlice.js
    cartSlice.js
  utils/
    formatDate.js
    validators.js
```

Downside: as the project grows, a change to one feature requires editing files across many folders.

### Feature-Based Structure (Recommended for Large Projects)

Groups everything related to a feature together. Makes it easy to find all code for a given feature and delete/add features without cross-cutting concerns.

```text
src/
  features/
    auth/
      components/
        LoginForm.jsx
        SignupForm.jsx
      hooks/
        useAuth.js
      store/
        authSlice.js
      services/
        authService.js
      index.js           ← public API for this feature
    cart/
      components/
        CartDrawer.jsx
        CartItem.jsx
      hooks/
        useCart.js
      store/
        cartSlice.js
  shared/
    components/
      Button.jsx
      Modal.jsx
    hooks/
      useLocalStorage.js
    utils/
      formatCurrency.js
  pages/
    Home.jsx
    Dashboard.jsx
  App.jsx
```

### Colocation Principle

Keep test files and style files next to the components they test/style. Do not centralize all tests in a `__tests__` folder at the root.

```text
features/
  auth/
    components/
      LoginForm.jsx
      LoginForm.test.jsx       ← colocated test
      LoginForm.module.css     ← colocated styles
```

---

## 4. State Management Strategy

### Decision Flow

```text
Does this state need to be shared with other components?
  No → useState / useReducer in the component

Does it need to be shared with siblings?
  Yes, and common parent is close → Lift state to parent

Does it need to be shared across many levels or unrelated components?
  Yes, and it's mostly UI state (theme, modals) → Context API
  Yes, and it's complex app state with many updates → Redux / Zustand

Does it come from the server?
  Yes → React Query / SWR / RTK Query (server state library)
```

### Layer Definitions

| Layer | Tool | Examples |
|---|---|---|
| Local UI state | useState / useReducer | Input value, toggle open, loading |
| Shared UI state | Context API | Theme, locale, auth user |
| Complex app state | Redux Toolkit / Zustand | Cart, multi-step forms, permissions |
| Server/remote state | React Query / SWR | User profiles, product lists, search results |
| URL state | React Router useSearchParams | Filters, pagination, current tab |

---

## 5. Data Flow Patterns

### Unidirectional Data Flow

React enforces a one-way data flow. Data flows down through props; events flow up through callbacks.

```text
Parent Component (owns state)
    │
    ↓  props (data flows down)
  Child A          Child B
    │
    ↓  props
  Grandchild
    │
    ↑  callback props (events bubble up)
  Child A
    │
    ↑  calls parent's handler
  Parent Component (updates state)
```

```jsx
// Data down, events up
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <Child
      count={count}                    // data down
      onIncrement={() => setCount(c => c + 1)}  // event up
    />
  );
}

function Child({ count, onIncrement }) {
  return (
    <div>
      <p>{count}</p>
      <button onClick={onIncrement}>+</button>
    </div>
  );
}
```

### Context vs Prop Drilling Decision

```text
Prop drilling is acceptable for 1–2 levels:
  App → Dashboard → UserCard (pass userId)

Context is appropriate for 3+ levels or cross-cutting concerns:
  App (ThemeContext) → Layout → Content → DeepWidget (reads theme)
```

---

## 6. Code Splitting Strategies

### Route-Based Splitting

The most impactful: users only load the code for the pages they visit.

```jsx
const Home = lazy(() => import("./pages/Home"));
const Dashboard = lazy(() => import("./pages/Dashboard"));
const AdminPanel = lazy(() => import("./pages/AdminPanel"));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/admin" element={<AdminPanel />} />
      </Routes>
    </Suspense>
  );
}
```

### Component-Based Splitting

For heavy components that are not immediately visible:

```jsx
// Heavy chart library — only load when needed
const DataChart = lazy(() => import("./components/DataChart"));

function Analytics() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <DataChart data={analyticsData} />
        </Suspense>
      )}
    </div>
  );
}
```

### Library Splitting

Separate large third-party libraries into their own chunks via dynamic import:

```jsx
// Lazy-load a heavy library only when the user triggers an action
async function handleExport() {
  const { generatePDF } = await import("./utils/pdfExporter");
  generatePDF(data);
}
```

### Bundle Analysis

Use tools to identify large dependencies:
- `webpack-bundle-analyzer`
- `vite-plugin-visualizer`
- Lighthouse in Chrome DevTools

---

## 7. Micro-Frontend Concepts

A micro-frontend architecture splits a large frontend application into independently deployable, independently developed pieces owned by separate teams.

### Key Characteristics

```text
Monolith Frontend:
  One large React app
  One team deploys all features
  All features share one dependency version

Micro-Frontend:
  Team A → /checkout feature (React 18)
  Team B → /catalog feature  (React 18)
  Team C → /account feature  (Vue or React 17)
  Shell app orchestrates them all
```

### Implementation Approaches

| Approach | Description | Tradeoff |
|---|---|---|
| Module Federation (Webpack 5) | Load remote modules at runtime | Most flexible, complex setup |
| iframes | Completely isolated pages in iframes | Maximum isolation, poor UX |
| Web Components | Framework-agnostic custom elements | Works across frameworks, limited React integration |
| Single-SPA | Meta-framework for multiple SPAs | Mature, framework-agnostic |
| NPM packages | Share components via private registry | Simple, but tight coupling on versions |

### Module Federation Example

```js
// catalog-app/webpack.config.js — exposes components
new ModuleFederationPlugin({
  name: "catalog",
  filename: "remoteEntry.js",
  exposes: {
    "./ProductList": "./src/components/ProductList",
  },
})

// shell-app — consumes from catalog
new ModuleFederationPlugin({
  remotes: {
    catalog: "catalog@http://catalog.example.com/remoteEntry.js",
  },
})

// shell-app component
const ProductList = lazy(() => import("catalog/ProductList"));
```

---

## 8. Server Components vs Client Components

### React Server Components (RSC) — React 18/19 with Next.js

Server Components render on the server and send serialized HTML/data to the client. They cannot use state, effects, or browser APIs.

```text
Server Component (runs on server, no JS sent to client):
  ✅ Can access databases directly
  ✅ Can access filesystem, secrets, environment variables
  ✅ Reduces client bundle size
  ❌ Cannot use useState, useEffect, useContext
  ❌ Cannot use browser APIs (window, localStorage)
  ❌ Cannot attach event listeners
  ❌ Cannot be interactive

Client Component (runs on client, has full React):
  ✅ Can use all hooks
  ✅ Can use browser APIs
  ✅ Can be interactive
  ❌ Code is sent to the client (bundle size)
  ❌ Cannot directly access server resources
```

### Next.js 13+ App Router

```jsx
// app/users/page.jsx — Server Component (default)
// No "use client" directive = server component
async function UsersPage() {
  // Direct database access — no API needed
  const users = await db.query("SELECT * FROM users");

  return (
    <div>
      <h1>Users</h1>
      {users.map(user => <UserCard key={user.id} user={user} />)}
    </div>
  );
}

// app/components/SearchBox.jsx — Client Component
"use client";  // marks as client component

function SearchBox({ onSearch }) {
  const [query, setQuery] = useState("");

  return (
    <input
      value={query}
      onChange={e => {
        setQuery(e.target.value);
        onSearch(e.target.value);
      }}
    />
  );
}
```

### Composition Rule

Client Components can import Server Components if they pass them as `children` or props. They cannot directly import and render Server Components inside them.

```jsx
// ✅ Correct — Server Component passes a Server Component as children
// ServerLayout.jsx (server)
async function ServerLayout({ children }) {
  const user = await getUser();
  return (
    <div>
      <Sidebar user={user} />   {/* server component */}
      {children}
    </div>
  );
}

// ❌ Wrong — Client Component imports Server Component
"use client";
import ServerOnlyComponent from "./ServerOnlyComponent"; // breaks
```

---

## 9. Rendering Patterns

### Client-Side Rendering (CSR)

```text
Browser → Request page
Server → Returns empty HTML + bundle.js
Browser → Downloads JS, runs React, renders content

Pros: Rich interactivity, good for authenticated dashboards
Cons: Slow initial load, poor SEO (empty initial HTML)
```

### Server-Side Rendering (SSR)

```text
Browser → Request page
Server → Runs React, generates full HTML, sends it
Browser → Displays HTML immediately (fast FCP)
Browser → Downloads JS, hydrates (attaches React)

Pros: Fast first contentful paint, SEO-friendly
Cons: Slower TTFB (server computation on every request), hydration mismatch issues
```

### Static Site Generation (SSG)

```text
Build time → Pre-renders all pages to HTML files
CDN → Serves static HTML files (very fast)
Browser → Displays immediately, then hydrates

Pros: Fastest possible load, no server needed, cached globally
Cons: Content is stale until next build, not suitable for personalized or real-time content
```

### Incremental Static Regeneration (ISR)

```text
Like SSG but pages are re-generated in the background at a configurable interval.

revalidate: 60 → page is regenerated at most once per minute

Pros: Near-static performance with fresh content
Cons: Content can be up to N seconds stale
```

### Comparison Table

| Pattern | Rendering | SEO | Dynamic Content | Example Use Case |
|---|---|---|---|---|
| CSR | Client | Poor | Excellent | Admin dashboards |
| SSR | Server (per request) | Excellent | Excellent | E-commerce product pages |
| SSG | Build time | Excellent | No (stale) | Marketing sites, blogs |
| ISR | Build time + background | Excellent | Near real-time | News sites, documentation |

---

## 10. Error Boundaries

An error boundary is a class component that catches JavaScript errors during rendering, in lifecycle methods, and in constructors of child components.

### Class Component Error Boundary

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log to error tracking service
    errorTracker.capture(error, { extra: errorInfo });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <DefaultErrorUI error={this.state.error} />;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary fallback={<p>Something went wrong.</p>}>
  <UserDashboard />
</ErrorBoundary>
```

### What Error Boundaries Do NOT Catch

```text
✅ Catches:
  - Errors in render methods
  - Errors in lifecycle methods
  - Errors in constructors of child components

❌ Does NOT catch:
  - Errors in event handlers (use try/catch)
  - Asynchronous errors (setTimeout, fetch callbacks)
  - Server-side rendering errors
  - Errors thrown in the error boundary itself
```

### Granular Error Boundaries

```jsx
// Global boundary — catches catastrophic errors
<ErrorBoundary fallback={<GlobalError />}>
  <App />
</ErrorBoundary>

// Feature-level boundary — isolates a feature, rest of app continues
<ErrorBoundary fallback={<WidgetError />}>
  <RecommendationWidget />
</ErrorBoundary>

// Component-level boundary — individual item errors don't crash the list
<ul>
  {items.map(item => (
    <ErrorBoundary key={item.id} fallback={<BrokenItemPlaceholder />}>
      <ExpensiveItem item={item} />
    </ErrorBoundary>
  ))}
</ul>
```

---

## 11. Suspense and Concurrent Features

### Suspense

`Suspense` lets components "wait" for something and show a fallback UI. It works with lazy-loaded components and data fetching (via suspense-compatible libraries).

```jsx
// With React.lazy
<Suspense fallback={<Spinner />}>
  <LazyLoadedComponent />
</Suspense>

// With data fetching (React Query v5 / Next.js)
<Suspense fallback={<UserSkeleton />}>
  <UserProfile userId={id} />
</Suspense>
```

### Streaming SSR with Suspense

In Next.js 13+ with App Router, Suspense boundaries enable streaming — the server streams HTML progressively, sending completed sections as they are ready.

```jsx
// The shell HTML is sent immediately
// UserProfile streams when ready
// ActivityFeed streams independently when ready
async function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<ProfileSkeleton />}>
        <UserProfile />           {/* streams when ready */}
      </Suspense>
      <Suspense fallback={<FeedSkeleton />}>
        <ActivityFeed />          {/* streams independently */}
      </Suspense>
    </div>
  );
}
```

### useTransition

`useTransition` marks state updates as non-urgent, allowing React to keep the UI responsive while the transition is in progress.

```jsx
function Search() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value);           // urgent — update input immediately

    startTransition(() => {
      // non-urgent — React can interrupt this if the user keeps typing
      setResults(performExpensiveSearch(e.target.value));
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultsList results={results} />}
    </div>
  );
}
```

---

## 12. Performance Patterns

### React.memo

Prevents re-renders when props have not changed (shallow comparison).

```jsx
const ExpensiveList = React.memo(function ExpensiveList({ items, onDelete }) {
  console.log("ExpensiveList renders");
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>
          {item.name}
          <button onClick={() => onDelete(item.id)}>Delete</button>
        </li>
      ))}
    </ul>
  );
});

// Parent must stabilize the onDelete callback, otherwise memo is useless
function Parent() {
  const [items, setItems] = useState(initialItems);

  // Without useCallback, onDelete is a new function reference on every render
  // That breaks React.memo's shallow comparison
  const handleDelete = useCallback((id) => {
    setItems(prev => prev.filter(item => item.id !== id));
  }, []);

  return <ExpensiveList items={items} onDelete={handleDelete} />;
}
```

### useMemo

Memoizes expensive computations.

```jsx
function ProductList({ products, filters }) {
  // Only recompute when products or filters change
  const filteredProducts = useMemo(
    () => products
      .filter(p => p.category === filters.category)
      .filter(p => p.price >= filters.minPrice && p.price <= filters.maxPrice)
      .sort((a, b) => a.price - b.price),
    [products, filters]
  );

  return <ul>{filteredProducts.map(p => <ProductItem key={p.id} product={p} />)}</ul>;
}
```

### Virtualization

Render only visible items in long lists. For 1000+ items, rendering all of them causes serious performance problems.

```jsx
import { FixedSizeList as List } from "react-window";

function VirtualList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <List
      height={600}       // visible container height
      itemCount={items.length}
      itemSize={50}      // height of each row
      width="100%"
    >
      {Row}
    </List>
  );
}
```

### When to Optimize

```text
Do NOT optimize prematurely.

Measure first:
  → React DevTools Profiler: find slow renders
  → Chrome Performance tab: find long tasks
  → Lighthouse: overall page performance

Apply in this order:
  1. Fix architectural issues (e.g., colocate state, avoid re-fetching)
  2. Virtualize long lists
  3. Code split large components
  4. Add React.memo + useCallback/useMemo where profiler shows repeated expensive renders
```

---

## 13. Testing Strategy

### Testing Pyramid

```text
          /\
         /  \  E2E Tests (few)
        /    \  Playwright, Cypress
       /------\
      /        \  Integration Tests (moderate)
     /  Testing  \  React Testing Library
    /   Library   \
   /--------------\
  /                \  Unit Tests (many)
 /    Jest + RTL    \  Utility functions, hooks
/____________________\
```

### Unit Tests

Test individual functions and hooks in isolation.

```jsx
// Testing a utility function
test("formatCurrency formats USD", () => {
  expect(formatCurrency(1234.56, "USD")).toBe("$1,234.56");
});

// Testing a custom hook
import { renderHook, act } from "@testing-library/react";

test("useCounter increments", () => {
  const { result } = renderHook(() => useCounter(0));
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});
```

### Integration Tests (Component Tests)

Test components with real user interactions. React Testing Library philosophy: test behavior, not implementation.

```jsx
import { render, screen, fireEvent } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

test("login form submits with user credentials", async () => {
  const mockSubmit = jest.fn();
  render(<LoginForm onSubmit={mockSubmit} />);

  await userEvent.type(screen.getByLabelText("Email"), "alice@test.com");
  await userEvent.type(screen.getByLabelText("Password"), "password123");
  await userEvent.click(screen.getByRole("button", { name: "Log In" }));

  expect(mockSubmit).toHaveBeenCalledWith({
    email: "alice@test.com",
    password: "password123",
  });
});
```

### E2E Tests

Test full user flows across the entire application.

```js
// Playwright example
test("user can complete checkout", async ({ page }) => {
  await page.goto("/products");
  await page.click("text=Add to Cart");
  await page.click("text=Checkout");
  await page.fill('[name="email"]', "test@example.com");
  await page.click("text=Place Order");
  await expect(page.locator("text=Order confirmed")).toBeVisible();
});
```

---

## 14. Common Architectural Mistakes

### Mistake 1: Mega-Component

```jsx
// ❌ Wrong — everything in one component
function Dashboard() {
  // 300 lines of state, effects, handlers, and JSX
}

// ✅ Correct — decompose by domain and responsibility
function Dashboard() {
  return (
    <div>
      <DashboardHeader />
      <DashboardMetrics />
      <RecentActivity />
    </div>
  );
}
```

### Mistake 2: Prop Drilling Beyond Two Levels

```jsx
// ❌ Wrong — username threaded through 4 levels
<App username={username}>
  <Layout username={username}>
    <Sidebar username={username}>
      <UserMenu username={username} />

// ✅ Correct — use Context or Zustand
const UserContext = createContext();
<UserContext.Provider value={{ username }}>
  <Layout>
    <Sidebar>
      <UserMenu />   {/* reads username from context */}
```

### Mistake 3: Storing Server State in Redux

```jsx
// ❌ Wrong — reinventing React Query with Redux
const userSlice = createSlice({
  name: "user",
  initialState: { data: null, loading: false, error: null },
  // endless loading/success/error action types...
});

// ✅ Correct — use a server state library
function UserProfile({ id }) {
  const { data: user, isLoading } = useQuery({
    queryKey: ["user", id],
    queryFn: () => fetchUser(id),
  });
}
```

### Mistake 4: Over-Contextualizing

Not every piece of shared state needs a Context. Contexts have a cost (all consumers re-render on any change). For small trees, prop drilling is fine.

### Mistake 5: No Error Boundaries

Without error boundaries, a single render error crashes the entire application. Every major feature boundary should have an error boundary.

---

## 15. Best Practices

### 1. Design at the Feature Level

Group code by domain feature, not by technical layer. A feature is a cohesive slice of business functionality.

### 2. Define Clear Public APIs for Features

Each feature should expose only what other features need via an `index.js` barrel file. Internal implementation details stay internal.

```js
// features/auth/index.js — public API
export { LoginForm } from "./components/LoginForm";
export { useAuth } from "./hooks/useAuth";
export { authReducer } from "./store/authSlice";
// Internal files are not exported
```

### 3. Keep Components Small

A component should ideally fit in one screen (~50-100 lines). If it is growing, look for natural split points.

### 4. Prefer Composition to Configuration

Rather than a component with many `mode`, `variant`, and `type` props that change its entire behavior, create separate focused components.

```jsx
// ❌ Over-configured
<Button variant="primary" size="large" loading={true} iconLeft="arrow" disabled />

// ✅ Composed
<IconButton icon={<ArrowIcon />} loading={loading} />
```

### 5. Write Accessible Components by Default

Use semantic HTML, proper ARIA attributes, keyboard navigation, and focus management from the start. Retrofitting accessibility is expensive.

### 6. Use TypeScript

TypeScript catches architectural errors at compile time, makes refactoring safe, and documents component APIs through types.

### 7. Establish a Render Budget

Profile regularly and set a performance budget. No route should take more than 200ms to become interactive.
