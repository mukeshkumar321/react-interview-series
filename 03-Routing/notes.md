# React Router v6

## Table of Contents

- [1. What is Client-Side Routing?](#1-what-is-client-side-routing)
- [2. Installation and Setup](#2-installation-and-setup)
- [3. Basic Routing: Routes and Route](#3-basic-routing-routes-and-route)
- [4. Link and NavLink](#4-link-and-navlink)
- [5. useNavigate Hook](#5-usenavigate-hook)
- [6. useParams Hook](#6-useparams-hook)
- [7. useSearchParams Hook](#7-usesearchparams-hook)
- [8. useLocation Hook](#8-uselocation-hook)
- [9. Nested Routes and Outlet](#9-nested-routes-and-outlet)
- [10. Layout Routes](#10-layout-routes)
- [11. Protected and Private Routes](#11-protected-and-private-routes)
- [12. Lazy Loading Routes](#12-lazy-loading-routes)
- [13. Error Boundaries in Routing](#13-error-boundaries-in-routing)
- [14. useRouteError Hook](#14-userouteerror-hook)
- [15. Navigate Component](#15-navigate-component)
- [16. Relative vs Absolute Paths](#16-relative-vs-absolute-paths)
- [17. v5 vs v6 Differences](#17-v5-vs-v6-differences)
- [18. Common Mistakes](#18-common-mistakes)
- [19. Best Practices](#19-best-practices)

---

## 1. What is Client-Side Routing?

### Server-Side Routing (Traditional)

In a traditional multi-page application, every URL change triggers a full HTTP request to the server. The server responds with a complete HTML document and the browser re-renders the entire page from scratch.

```text
User clicks link
      ↓
Browser sends HTTP GET /about
      ↓
Server processes request
      ↓
Server returns full HTML page
      ↓
Browser re-renders entire page
      ↓
All state is lost, scroll position resets
```

### Client-Side Routing

In a Single Page Application (SPA), JavaScript intercepts URL changes and renders different components without making a full page reload. The URL changes in the browser bar, the back/forward buttons work, but no round-trip to the server occurs for the HTML.

```text
User clicks <Link to="/about">
      ↓
React Router intercepts the click
      ↓
URL updates via History API (pushState)
      ↓
React Router matches new URL to routes
      ↓
React renders the matched component
      ↓
State is preserved, no page reload
```

### Why Client-Side Routing is Needed

| Concern | Without CSR | With CSR |
|---|---|---|
| Navigation speed | Full page reload | Instant (no network) |
| State preservation | Lost on every route change | Maintained |
| User experience | Flash of blank page | Smooth transitions |
| Animations | Very difficult | Easy |
| Shared layout (header/nav) | Must be re-rendered | Stays mounted |

### The History API

React Router is built on the browser's `History API`. Two key methods:

- `history.pushState(state, title, url)` — adds a new entry to the history stack
- `history.replaceState(state, title, url)` — replaces the current entry without adding to the stack

React Router wraps this API so you rarely interact with it directly.

---

## 2. Installation and Setup

### Installation

```bash
npm install react-router-dom
```

React Router v6 requires React 16.8 or higher (hooks support).

### BrowserRouter

`BrowserRouter` uses the HTML5 History API. It is the standard choice for web applications deployed to a server that can handle dynamic URLs.

```jsx
import { BrowserRouter } from "react-router-dom";
import { createRoot } from "react-dom/client";

createRoot(document.getElementById("root")).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

### HashRouter

`HashRouter` uses the URL hash (`#`) for routing. It works on static file servers that cannot handle dynamic URLs (like GitHub Pages with no fallback config).

```jsx
// URL looks like: https://example.com/#/about
import { HashRouter } from "react-router-dom";
```

### MemoryRouter

`MemoryRouter` keeps the URL in memory, not in the browser bar. Useful for testing and React Native.

```jsx
import { MemoryRouter } from "react-router-dom";
// No URL changes visible to the user
```

### createBrowserRouter (v6.4+)

React Router 6.4 introduced data APIs (`loader`, `action`, `errorElement`). These require `createBrowserRouter`.

```jsx
import { createBrowserRouter, RouterProvider } from "react-router-dom";

const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <ErrorPage />,
    children: [
      { path: "about", element: <About /> },
      { path: "users/:id", element: <UserDetail /> },
    ],
  },
]);

createRoot(document.getElementById("root")).render(
  <RouterProvider router={router} />
);
```

---

## 3. Basic Routing: Routes and Route

### Routes Component

`Routes` looks at the current URL and renders the first `Route` whose `path` matches. Only one route is rendered at a time (unlike v5's `Switch`-less rendering where all matching routes could render).

```jsx
import { Routes, Route } from "react-router-dom";

function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/about" element={<About />} />
      <Route path="/contact" element={<Contact />} />
    </Routes>
  );
}
```

### Route Props

| Prop | Purpose |
|---|---|
| `path` | URL pattern to match |
| `element` | JSX to render when matched |
| `index` | Mark as the default child route |

### Path Matching

React Router v6 uses ranked matching — the most specific route wins, regardless of declaration order.

```jsx
// These routes are matched by specificity, not order
<Routes>
  <Route path="/users/new" element={<NewUser />} />  {/* More specific */}
  <Route path="/users/:id" element={<UserDetail />} /> {/* Less specific */}
  <Route path="/users" element={<UserList />} />
</Routes>
```

### Exact Matching (Always On in v6)

In v6, all routes match exactly by default. The `exact` prop from v5 is removed. The path `/about` only matches the URL `/about`, not `/about/team`.

```jsx
// v6 — exact is the default behavior
<Route path="/about" element={<About />} />

// v5 — you had to add exact
<Route exact path="/about" component={About} />
```

### Wildcard Route (Catch-All)

Use `*` to match any remaining path segments. Commonly used for 404 pages.

```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
  <Route path="*" element={<NotFound />} />
</Routes>
```

### Index Routes

An index route is the default child rendered when the parent route matches but no child route matches. It uses the `index` prop instead of a `path`.

```jsx
<Routes>
  <Route path="/dashboard" element={<Dashboard />}>
    <Route index element={<DashboardHome />} />  {/* renders at /dashboard */}
    <Route path="stats" element={<Stats />} />   {/* renders at /dashboard/stats */}
  </Route>
</Routes>
```

---

## 4. Link and NavLink

### Link

`Link` renders an `<a>` tag but prevents the default browser navigation, allowing React Router to handle the transition.

```jsx
import { Link } from "react-router-dom";

function Nav() {
  return (
    <nav>
      <Link to="/">Home</Link>
      <Link to="/about">About</Link>
      <Link to="/contact">Contact</Link>
    </nav>
  );
}
```

### Link with State

You can pass state data alongside navigation, accessible via `useLocation()`:

```jsx
<Link to="/profile" state={{ from: "dashboard", userId: 42 }}>
  Go to Profile
</Link>
```

### Link with Replace

Use `replace` to replace the current history entry instead of adding a new one:

```jsx
<Link to="/login" replace>Log In</Link>
```

### NavLink

`NavLink` is like `Link` but adds styling capabilities for the active route. React Router automatically applies an `active` CSS class when the link's `to` path matches the current URL.

```jsx
import { NavLink } from "react-router-dom";

function Nav() {
  return (
    <nav>
      <NavLink to="/">Home</NavLink>
      <NavLink to="/about">About</NavLink>
    </nav>
  );
}
```

Default CSS class behavior:

```css
/* React Router adds this automatically */
a.active {
  font-weight: bold;
  color: blue;
}
```

### NavLink className Function

For dynamic class names, pass a function that receives `{ isActive, isPending }`:

```jsx
<NavLink
  to="/about"
  className={({ isActive }) =>
    isActive ? "nav-link nav-link--active" : "nav-link"
  }
>
  About
</NavLink>
```

### NavLink style Function

Same pattern works for inline styles:

```jsx
<NavLink
  to="/about"
  style={({ isActive }) => ({
    color: isActive ? "blue" : "inherit",
    fontWeight: isActive ? "bold" : "normal",
  })}
>
  About
</NavLink>
```

### NavLink end Prop

By default, `/` (home) is considered active for all routes because every URL starts with `/`. Use `end` to only activate the link when the URL matches exactly.

```jsx
// Without end, this link is always active
<NavLink to="/">Home</NavLink>

// With end, only active at exactly "/"
<NavLink to="/" end>Home</NavLink>
```

---

## 5. useNavigate Hook

`useNavigate` returns a function for programmatic navigation. It replaces `useHistory` from v5.

### Basic Navigation

```jsx
import { useNavigate } from "react-router-dom";

function LoginForm() {
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    await login(credentials);
    navigate("/dashboard");  // navigate to absolute path
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

### navigate(to, options)

```jsx
// Push a new entry (default)
navigate("/about");

// Replace current entry (no back button entry added)
navigate("/login", { replace: true });

// Pass state
navigate("/profile", { state: { from: "/checkout" } });

// Go back one page (like browser back button)
navigate(-1);

// Go forward one page
navigate(1);

// Go back two pages
navigate(-2);
```

### replace vs push

```text
History Stack Example:
  Home → About → Contact  (current: Contact)

navigate("/login")          → Home → About → Contact → Login   (can go back to Contact)
navigate("/login", { replace: true }) → Home → About → Login  (Contact is removed)
```

Use `replace: true` when you do not want the user to navigate back to the previous page. Common use cases:
- After login (remove the login page from history)
- After form submission
- Redirect from a deprecated URL

### navigate(-1) Deep Dive

```jsx
// This is equivalent to clicking the browser back button
navigate(-1);

// CAUTION: if there is no history (e.g. user opened the page directly),
// navigate(-1) will leave the app (go to the previous browser page or close)
// Always guard this:
function BackButton() {
  const navigate = useNavigate();
  const location = useLocation();

  const canGoBack = window.history.length > 1;

  return (
    <button onClick={() => canGoBack ? navigate(-1) : navigate("/")}>
      Back
    </button>
  );
}
```

### useNavigate in useEffect

```jsx
// Redirect after data loads
function Dashboard() {
  const navigate = useNavigate();
  const { user, loading } = useAuth();

  useEffect(() => {
    if (!loading && !user) {
      navigate("/login", { replace: true });
    }
  }, [user, loading, navigate]);

  if (loading) return <Spinner />;
  return <DashboardContent />;
}
```

---

## 6. useParams Hook

`useParams` returns an object of URL parameters from the dynamic segments in the route's path.

### Basic Usage

```jsx
import { useParams } from "react-router-dom";

// Route definition: <Route path="/users/:userId" element={<UserDetail />} />

function UserDetail() {
  const { userId } = useParams();

  return <p>User ID: {userId}</p>;
}
// URL: /users/42  →  userId = "42"
```

### Multiple Params

```jsx
// Route: /teams/:teamId/members/:memberId
function MemberDetail() {
  const { teamId, memberId } = useParams();

  return (
    <p>
      Team: {teamId}, Member: {memberId}
    </p>
  );
}
// URL: /teams/react/members/john  →  teamId = "react", memberId = "john"
```

### Params are Always Strings

URL parameters are always strings regardless of what you put in the URL.

```jsx
function Post() {
  const { id } = useParams();

  // id is "42" (string), not 42 (number)
  const numId = Number(id);  // convert when needed

  const post = posts.find(p => p.id === numId);
  // ...
}
```

### Optional Params (v6.4+)

```jsx
// Path: /profile/:username?
// Matches: /profile  AND  /profile/alice
function Profile() {
  const { username } = useParams();
  // username is undefined when at /profile
  // username is "alice" when at /profile/alice
}
```

---

## 7. useSearchParams Hook

`useSearchParams` reads and writes query string parameters. It returns a pair like `useState`: the current search params and a setter function.

### Reading Search Params

```jsx
import { useSearchParams } from "react-router-dom";

// URL: /products?category=electronics&sort=price&page=2
function ProductList() {
  const [searchParams] = useSearchParams();

  const category = searchParams.get("category");  // "electronics"
  const sort = searchParams.get("sort");           // "price"
  const page = Number(searchParams.get("page"));   // 2
  const missing = searchParams.get("q");           // null (not present)

  return <div>Category: {category}</div>;
}
```

### Writing Search Params

```jsx
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();

  const handleCategoryChange = (category) => {
    setSearchParams({ category, page: "1" });
    // URL becomes: /products?category=electronics&page=1
  };

  const handlePageChange = (page) => {
    // Preserve other params, only update page
    setSearchParams(prev => {
      prev.set("page", page);
      return prev;
    });
  };

  return (
    <div>
      <button onClick={() => handleCategoryChange("electronics")}>
        Electronics
      </button>
    </div>
  );
}
```

### URLSearchParams Methods

`searchParams` is a `URLSearchParams` instance with these key methods:

| Method | Description |
|---|---|
| `.get(key)` | Get the first value for a key, or `null` |
| `.getAll(key)` | Get all values for a key as an array |
| `.has(key)` | Check if a key exists |
| `.set(key, value)` | Set a value (replaces existing) |
| `.append(key, value)` | Add a value (keeps existing) |
| `.delete(key)` | Remove a key |
| `.toString()` | Serialize to query string |

### Multi-Value Params

```jsx
// URL: /products?tag=react&tag=hooks&tag=router
function ProductList() {
  const [searchParams] = useSearchParams();

  const tags = searchParams.getAll("tag");  // ["react", "hooks", "router"]

  return (
    <ul>
      {tags.map(tag => <li key={tag}>{tag}</li>)}
    </ul>
  );
}
```

---

## 8. useLocation Hook

`useLocation` returns the current location object. It is useful for reading the current URL, accessing passed state, or triggering effects on navigation.

### Location Object Shape

```text
{
  pathname: "/about",        // the URL path
  search:   "?q=react",     // the query string (including ?)
  hash:     "#section-2",   // the hash (including #)
  state:    { from: "/" },  // state passed via navigate() or <Link state={...}>
  key:      "default"       // unique key for this location
}
```

### Reading Pathname

```jsx
import { useLocation } from "react-router-dom";

function Breadcrumb() {
  const location = useLocation();

  const segments = location.pathname.split("/").filter(Boolean);

  return (
    <nav>
      {segments.map((segment, i) => (
        <span key={i}>{segment} / </span>
      ))}
    </nav>
  );
}
```

### Reading Location State

```jsx
// On the source page
navigate("/checkout", { state: { cartTotal: 99.99, itemCount: 3 } });

// Or via Link
<Link to="/checkout" state={{ cartTotal: 99.99 }}>Proceed</Link>

// On the destination page
function Checkout() {
  const location = useLocation();
  const { cartTotal, itemCount } = location.state || {};

  return <p>Total: ${cartTotal}</p>;
}
```

### "Redirect Back" Pattern

A common pattern: remember where the user came from, redirect to login, then redirect back after login.

```jsx
// ProtectedRoute — saves current location
function ProtectedRoute({ children }) {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  return children;
}

// LoginPage — reads saved location
function Login() {
  const navigate = useNavigate();
  const location = useLocation();

  const from = location.state?.from?.pathname || "/dashboard";

  const handleLogin = async () => {
    await loginUser();
    navigate(from, { replace: true });
  };

  return <button onClick={handleLogin}>Login</button>;
}
```

### useEffect on Route Change

```jsx
function App() {
  const location = useLocation();

  useEffect(() => {
    // Track page views on every route change
    analytics.track("page_view", { path: location.pathname });
  }, [location]);

  return <Routes>...</Routes>;
}
```

---

## 9. Nested Routes and Outlet

### What are Nested Routes?

Nested routes allow a parent route to render its own layout while child routes render into a designated slot within it. This is essential for layouts like dashboards, admin panels, and tabbed interfaces.

### Outlet Component

`Outlet` is a placeholder component that renders the currently active child route. Without an `Outlet`, child routes will match the URL but their elements will not appear on screen.

```jsx
import { Routes, Route, Outlet } from "react-router-dom";

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <nav>
        <Link to="overview">Overview</Link>
        <Link to="stats">Stats</Link>
        <Link to="settings">Settings</Link>
      </nav>

      {/* Child routes render here */}
      <Outlet />
    </div>
  );
}

function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={<Dashboard />}>
        <Route index element={<Overview />} />        {/* /dashboard */}
        <Route path="stats" element={<Stats />} />    {/* /dashboard/stats */}
        <Route path="settings" element={<Settings />} /> {/* /dashboard/settings */}
      </Route>
    </Routes>
  );
}
```

### How Outlet Renders

```text
URL: /dashboard/stats

Component Tree:
  <Dashboard>           ← matched by path="/dashboard"
    <h1>Dashboard</h1>
    <nav>...</nav>
    <Outlet />          ← replaced with:
      <Stats />         ← matched by path="stats"
  </Dashboard>
```

### Outlet Context

You can pass data from a parent layout to any nested child via `useOutletContext`:

```jsx
function Dashboard() {
  const [user] = useState({ name: "Alice" });

  return (
    <div>
      <h1>Dashboard</h1>
      <Outlet context={{ user }} />
    </div>
  );
}

// In any child route component:
import { useOutletContext } from "react-router-dom";

function Stats() {
  const { user } = useOutletContext();
  return <p>Stats for: {user.name}</p>;
}
```

---

## 10. Layout Routes

A layout route is a route that renders a shared UI wrapper and contains child routes — but its path does not add any URL segment. The children are rendered directly inside the layout.

### Without Path (Pathless Layout Route)

```jsx
function AppLayout() {
  return (
    <div>
      <Header />
      <Sidebar />
      <main>
        <Outlet />    {/* child routes render here */}
      </main>
      <Footer />
    </div>
  );
}

function App() {
  return (
    <Routes>
      {/* Layout route — no path, wraps child routes */}
      <Route element={<AppLayout />}>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/contact" element={<Contact />} />
      </Route>

      {/* Route without the layout — e.g. a full-screen login page */}
      <Route path="/login" element={<Login />} />
    </Routes>
  );
}
```

### Multiple Layouts

```jsx
function App() {
  return (
    <Routes>
      {/* Public layout */}
      <Route element={<PublicLayout />}>
        <Route path="/" element={<Home />} />
        <Route path="/pricing" element={<Pricing />} />
      </Route>

      {/* Dashboard layout (authenticated) */}
      <Route element={<DashboardLayout />}>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Route>

      {/* No layout */}
      <Route path="/login" element={<Login />} />
    </Routes>
  );
}
```

---

## 11. Protected and Private Routes

A protected route guards access to certain pages based on authentication or authorization. If the user is not allowed, they are redirected.

### Basic Protected Route

```jsx
import { Navigate, useLocation } from "react-router-dom";

function ProtectedRoute({ children }) {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();

  // Show loading spinner while checking auth
  if (isLoading) return <Spinner />;

  // Redirect to login, saving the current location so we can return after login
  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return children;
}

// Usage in routes
function App() {
  return (
    <Routes>
      <Route path="/login" element={<Login />} />
      <Route
        path="/dashboard"
        element={
          <ProtectedRoute>
            <Dashboard />
          </ProtectedRoute>
        }
      />
    </Routes>
  );
}
```

### Role-Based Protected Route

```jsx
function RequireRole({ allowedRoles, children }) {
  const { user, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) return <Spinner />;

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  if (!allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return children;
}

// Usage
<Route
  path="/admin"
  element={
    <RequireRole allowedRoles={["admin", "superadmin"]}>
      <AdminPanel />
    </RequireRole>
  }
/>
```

### Layout-Based Protected Route

Combine with layout routes for clean organization:

```jsx
function AuthLayout() {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

function App() {
  return (
    <Routes>
      <Route path="/login" element={<Login />} />

      {/* All routes under this layout are protected */}
      <Route element={<AuthLayout />}>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
        <Route path="/settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}
```

---

## 12. Lazy Loading Routes

Code splitting allows you to split your JavaScript bundle into smaller chunks that load on demand. React Router is the natural boundary for splitting because each route is an independent page.

### React.lazy and Suspense

```jsx
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

// Each import() creates a separate chunk
const Home = lazy(() => import("./pages/Home"));
const About = lazy(() => import("./pages/About"));
const Dashboard = lazy(() => import("./pages/Dashboard"));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

### Named Exports with lazy

`React.lazy` requires a default export. If the component uses a named export, create a wrapper:

```jsx
// Component file uses named export
export function UserSettings() { ... }

// Import using a re-export trick
const UserSettings = lazy(() =>
  import("./pages/UserSettings").then(module => ({
    default: module.UserSettings,
  }))
);
```

### Granular Suspense Boundaries

Place `Suspense` close to the lazy component for better UX:

```jsx
function App() {
  return (
    <Routes>
      <Route path="/" element={<Home />} />
      <Route
        path="/dashboard"
        element={
          <Suspense fallback={<DashboardSkeleton />}>
            <Dashboard />
          </Suspense>
        }
      />
    </Routes>
  );
}
```

### Bundle Impact

```text
Without code splitting:
  bundle.js   →  2.5 MB  (loads on first visit, all routes)

With code splitting per route:
  main.js     →  400 KB  (core app, always loaded)
  home.js     →  80 KB   (loaded when visiting /)
  dashboard.js → 300 KB  (loaded when visiting /dashboard)
  admin.js    →  200 KB  (loaded only if admin visits /admin)
```

---

## 13. Error Boundaries in Routing

React Router 6.4+ introduced route-level error handling with `errorElement`. When a loader, action, or the component itself throws an error, React Router renders the `errorElement` instead.

### errorElement on a Route

```jsx
const router = createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <GlobalErrorPage />,  // catches errors from any child
    children: [
      {
        path: "users/:id",
        element: <UserDetail />,
        errorElement: <UserErrorPage />,  // catches errors only from this route
        loader: async ({ params }) => {
          const user = await fetchUser(params.id);
          if (!user) throw new Response("Not Found", { status: 404 });
          return user;
        },
      },
    ],
  },
]);
```

### Error Boundary Hierarchy

```text
createBrowserRouter([
  {
    path: "/",
    element: <Root />,
    errorElement: <RootError />,       ← catches any unhandled error in the tree
    children: [
      {
        path: "users",
        element: <Users />,
        errorElement: <UsersError />,  ← catches errors from /users and its children
        children: [
          {
            path: ":id",
            element: <UserDetail />,
            errorElement: <UserDetailError />,  ← most specific
          }
        ]
      }
    ]
  }
])
```

---

## 14. useRouteError Hook

`useRouteError` is used inside an `errorElement` component to access the thrown error or Response.

```jsx
import { useRouteError, isRouteErrorResponse } from "react-router-dom";

function ErrorPage() {
  const error = useRouteError();

  // Check if it's an HTTP Response error (thrown from loader/action)
  if (isRouteErrorResponse(error)) {
    return (
      <div>
        <h1>{error.status} {error.statusText}</h1>
        {error.status === 404 && <p>Page not found.</p>}
        {error.status === 403 && <p>You are not authorized.</p>}
      </div>
    );
  }

  // Regular JavaScript error (thrown from component or loader)
  return (
    <div>
      <h1>Oops! Something went wrong.</h1>
      <p>{error.message}</p>
    </div>
  );
}
```

### Throwing Responses from Loaders

```jsx
async function userLoader({ params }) {
  const res = await fetch(`/api/users/${params.id}`);

  if (res.status === 404) {
    throw new Response("User Not Found", { status: 404 });
  }

  if (!res.ok) {
    throw new Response("Server Error", { status: 500 });
  }

  return res.json();
}
```

---

## 15. Navigate Component

`Navigate` is the declarative equivalent of `useNavigate`. It renders nothing but causes a navigation when it mounts.

### Basic Redirect

```jsx
import { Navigate } from "react-router-dom";

function OldPage() {
  // Renders nothing, immediately navigates to /new-page
  return <Navigate to="/new-page" replace />;
}
```

### Conditional Redirect

```jsx
function Dashboard() {
  const { user } = useAuth();

  if (!user) {
    return <Navigate to="/login" replace />;
  }

  return <DashboardContent />;
}
```

### Navigate vs useNavigate

| | `Navigate` component | `useNavigate` hook |
|---|---|---|
| Type | Declarative (JSX) | Imperative (function call) |
| When it runs | On render | On explicit call |
| Use case | Conditional render logic | Event handlers, side effects |

```jsx
// Declarative — conditional in JSX
if (!user) return <Navigate to="/login" replace />;

// Imperative — inside event handler
const navigate = useNavigate();
const handleLogout = () => {
  logout();
  navigate("/login", { replace: true });
};
```

---

## 16. Relative vs Absolute Paths

### Absolute Paths

An absolute path starts with `/`. It always navigates to that exact URL from the root.

```jsx
// Always navigates to /about regardless of current location
<Link to="/about">About</Link>
navigate("/about");
```

### Relative Paths

A relative path without a leading `/` is resolved relative to the nearest parent route's path.

```jsx
// Inside a component rendered at /dashboard
<Link to="stats">Stats</Link>     // → /dashboard/stats
<Link to="../home">Home</Link>    // → /home (go up one level)
<Link to=".">Current</Link>       // → /dashboard (self)
```

### Relative Path Example

```jsx
// Route: /dashboard/*
function Dashboard() {
  return (
    <div>
      {/* These are relative to /dashboard */}
      <Link to="overview">Overview</Link>   {/* /dashboard/overview */}
      <Link to="settings">Settings</Link>  {/* /dashboard/settings */}

      {/* Absolute path */}
      <Link to="/login">Login</Link>       {/* /login */}
    </div>
  );
}
```

### useNavigate with Relative Paths

```jsx
const navigate = useNavigate();

// relative (default) — relative to current route
navigate("details");      // e.g., from /users → /users/details

// absolute
navigate("/users/details");

// relative: "path" mode — relative to current URL path segment
navigate("..", { relative: "path" });
```

---

## 17. v5 vs v6 Differences

### Migration Summary

| Feature | v5 | v6 |
|---|---|---|
| Match all routes | `<Switch>` | `<Routes>` |
| Exact matching | `exact` prop required | Always exact by default |
| Nested routes | Defined at render site | Defined in parent Route's children |
| Child rendering | N/A | `<Outlet />` required |
| Redirect | `<Redirect>` | `<Navigate>` |
| Programmatic nav | `useHistory()` | `useNavigate()` |
| Route config | Component-based | Also supports data API (6.4+) |
| Ranking | First match wins | Most specific match wins |

### Nested Routes: v5 vs v6

```jsx
// v5 — nested routes defined at the render site inside child components
// App.js
function App() {
  return (
    <Switch>
      <Route path="/dashboard" component={Dashboard} />
    </Switch>
  );
}

// Dashboard.js — must know its own path prefix
function Dashboard({ match }) {
  return (
    <div>
      <Switch>
        <Route path={`${match.path}/stats`} component={Stats} />
        <Route path={`${match.path}/settings`} component={Settings} />
      </Switch>
    </div>
  );
}

// v6 — nested routes centralized in parent, child uses <Outlet>
function App() {
  return (
    <Routes>
      <Route path="/dashboard" element={<Dashboard />}>
        <Route path="stats" element={<Stats />} />
        <Route path="settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}

function Dashboard() {
  return (
    <div>
      <Outlet />   {/* Stats or Settings renders here */}
    </div>
  );
}
```

### Route Ranking: v5 vs v6

```jsx
// v5 — order matters, first match wins
<Switch>
  <Route path="/users/new" component={NewUser} />   {/* must come first */}
  <Route path="/users/:id" component={UserDetail} />
</Switch>

// v6 — ranking by specificity, order does not matter
<Routes>
  <Route path="/users/:id" element={<UserDetail />} />  {/* can come first */}
  <Route path="/users/new" element={<NewUser />} />     {/* still matches /users/new correctly */}
</Routes>
```

---

## 18. Common Mistakes

### Mistake 1: Forgetting Outlet in Layout Components

```jsx
// ❌ Wrong — child routes match but render nothing
function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      {/* No Outlet! Children never render */}
    </div>
  );
}

// ✅ Correct
function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Outlet />
    </div>
  );
}
```

### Mistake 2: Using navigate() Before Component Mounts

```jsx
// ❌ Wrong — calling navigate during render phase
function App() {
  const navigate = useNavigate();
  navigate("/login");  // This causes an infinite loop or React warning
  return null;
}

// ✅ Correct — use Navigate component or navigate inside useEffect
function App() {
  return <Navigate to="/login" replace />;
}
```

### Mistake 3: Missing NavLink end on Root Path

```jsx
// ❌ Wrong — "/" link is always active because every URL starts with "/"
<NavLink to="/">Home</NavLink>

// ✅ Correct
<NavLink to="/" end>Home</NavLink>
```

### Mistake 4: Nested path with Leading Slash

```jsx
// ❌ Wrong — absolute path bypasses parent route context
<Route path="/dashboard" element={<Dashboard />}>
  <Route path="/stats" element={<Stats />} />  {/* This matches /stats, NOT /dashboard/stats */}
</Route>

// ✅ Correct — relative path (no leading /)
<Route path="/dashboard" element={<Dashboard />}>
  <Route path="stats" element={<Stats />} />   {/* Matches /dashboard/stats */}
</Route>
```

### Mistake 5: Not Handling Null State from useLocation

```jsx
// ❌ Wrong — crashes if state is null (user navigated directly)
function Checkout() {
  const location = useLocation();
  const { from } = location.state;  // TypeError if state is null
}

// ✅ Correct — always provide a default
function Checkout() {
  const location = useLocation();
  const { from } = location.state || { from: "/" };
}
```

### Mistake 6: params as Numbers

```jsx
// ❌ Wrong — comparing string param to number
const { id } = useParams();
const post = posts.find(p => p.id === id);  // id is "5", p.id is 5 → never matches

// ✅ Correct — convert param
const post = posts.find(p => p.id === Number(id));
```

---

## 19. Best Practices

### 1. Centralize Route Definitions

Define all routes in a single file or configuration object to make them easier to audit and update.

```jsx
// routes.jsx
export const routes = [
  { path: "/", element: <Home /> },
  { path: "/about", element: <About /> },
  { path: "/users/:id", element: <UserDetail /> },
  { path: "*", element: <NotFound /> },
];

// App.jsx
import { routes } from "./routes";

function App() {
  return (
    <Routes>
      {routes.map(route => (
        <Route key={route.path} path={route.path} element={route.element} />
      ))}
    </Routes>
  );
}
```

### 2. Lazy-Load Every Route

Split every page-level component into its own bundle. The performance benefit is immediate on large applications.

```jsx
const Home = lazy(() => import("./pages/Home"));
const About = lazy(() => import("./pages/About"));
```

### 3. Provide a 404 Route

```jsx
<Routes>
  {/* ... other routes ... */}
  <Route path="*" element={<NotFound />} />
</Routes>
```

### 4. Use Replace on Post-Auth Redirect

After a successful login, use `replace: true` so pressing the back button does not return to the login page.

```jsx
navigate(from, { replace: true });
```

### 5. Use NavLink for Navigation Menus

Always use `NavLink` instead of `Link` for navigation menus that benefit from active state styling.

### 6. Guard Against Missing State

```jsx
const from = location.state?.from?.pathname ?? "/dashboard";
```

### 7. Co-locate Route Segments

Place route-level files near their associated routes to make the project navigable.

```text
src/
  pages/
    Home/
      index.jsx
      Home.test.jsx
    Dashboard/
      index.jsx
      Dashboard.test.jsx
      components/
        Sidebar.jsx
```

### 8. Type Your Params with TypeScript

```tsx
import { useParams } from "react-router-dom";

type UserParams = {
  userId: string;
};

function UserDetail() {
  const { userId } = useParams<UserParams>();
  // userId is typed as string | undefined
}
```
