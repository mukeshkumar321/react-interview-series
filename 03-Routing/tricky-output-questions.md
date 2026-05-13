## React Router v6 — Tricky Output Questions

> These questions test your understanding of route matching order, navigation behavior, param types, nested route rendering with Outlet, and edge cases around useLocation state. Each scenario reflects a real interview or debugging situation.

---

## 1. Route Matching

### Q1

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/users/:id" element={<span>Dynamic</span>} />
        <Route path="/users/new" element={<span>New User</span>} />
      </Routes>
    </BrowserRouter>
  );
}
```

#### ❓ The URL is `/users/new`. Which element renders?

<details>
<summary>✅ Answer</summary>

```txt
New User
```

**Explanation:** React Router v6 uses ranked/scored matching. A static segment like `new` is ranked higher than a dynamic segment like `:id`. So `/users/new` always renders the "New User" route, regardless of declaration order. In v5 with Switch, the first matching route won — declaration order mattered. In v6, specificity wins.

</details>

---

### Q2

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/about" element={<span>About</span>} />
        <Route path="/about/team" element={<span>Team</span>} />
        <Route path="*" element={<span>404</span>} />
      </Routes>
    </BrowserRouter>
  );
}
```

#### ❓ What renders for each URL: `/about`, `/about/team`, `/about/team/members`?

<details>
<summary>✅ Answer</summary>

```txt
/about              → About
/about/team         → Team
/about/team/members → 404
```

**Explanation:** v6 routes match exactly by default. `/about` only matches `/about`. `/about/team` only matches `/about/team`. `/about/team/members` does not match either static route, so the `*` wildcard catches it. In v5, `/about` would match all three because it did not have `exact` by default.

</details>

---

### Q3

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="*" element={<span>Wildcard</span>} />
        <Route path="/" element={<span>Home</span>} />
        <Route path="/about" element={<span>About</span>} />
      </Routes>
    </BrowserRouter>
  );
}
```

#### ❓ The URL is `/about`. What renders?

<details>
<summary>✅ Answer</summary>

```txt
About
```

**Explanation:** React Router v6 uses ranked matching — it ignores declaration order and picks the most specific match. The `*` wildcard has the lowest rank. `/about` is an exact static match and wins over `*`. The order of `<Route>` elements in JSX does not matter for v6 matching.

</details>

---

### Q4

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/products" element={<ProductList />} />
        <Route path="/products/:category/:id" element={<ProductDetail />} />
        <Route path="/products/:id" element={<SingleProduct />} />
      </Routes>
    </BrowserRouter>
  );
}
```

#### ❓ Which route matches `/products/electronics/42`?

<details>
<summary>✅ Answer</summary>

```txt
ProductDetail (path="/products/:category/:id")
```

**Explanation:** React Router v6 matches the route with the most matching segments. `/products/:category/:id` has three segments and matches `/products/electronics/42` exactly. `/products/:id` only has two segments and cannot match a three-segment URL. The number of static and dynamic segments in the pattern must match the number in the actual URL.

</details>

---

### Q5

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />}>
          <Route path="stats" element={<Stats />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
    </div>
  );
}
```

#### ❓ The URL is `/dashboard/stats`. What renders on screen?

<details>
<summary>✅ Answer</summary>

```txt
Dashboard heading only — Stats does NOT appear
```

**Explanation:** Even though `/dashboard/stats` matches the `stats` child route, the `Dashboard` component does not include an `<Outlet />`. The `<Outlet />` is the slot where child routes render their elements. Without it, child route elements are matched but silently discarded. Adding `<Outlet />` inside `Dashboard` would make `Stats` appear.

</details>

---

## 2. Navigation

### Q6

```jsx
function App() {
  const navigate = useNavigate();

  return (
    <button onClick={() => {
      navigate("/step2");
      navigate("/step3");
    }}>
      Go
    </button>
  );
}
```

#### ❓ After clicking the button, what is the final URL and how many entries are in the history stack?

<details>
<summary>✅ Answer</summary>

```txt
Final URL: /step3
History stack entries added: 2 (one for /step2, one for /step3)
```

**Explanation:** Each `navigate()` call pushes a new entry onto the browser history stack. Both calls execute synchronously. The browser ends up at `/step3` (the last navigate call wins as the final URL), but `/step2` is still in the history stack — pressing back will go to `/step2`, then back again will go to the original page.

</details>

---

### Q7

```jsx
function Login() {
  const navigate = useNavigate();

  const handleLogin = () => {
    navigate("/dashboard");
    navigate("/dashboard", { replace: true });
  };

  return <button onClick={handleLogin}>Login</button>;
}
```

#### ❓ What is in the history stack after clicking the button?

<details>
<summary>✅ Answer</summary>

```txt
The history stack has /dashboard once, not twice.
The /login page entry is gone.
```

**Explanation:** The first `navigate("/dashboard")` pushes `/dashboard` onto the history stack. The second `navigate("/dashboard", { replace: true })` replaces the current entry (now `/dashboard`) with `/dashboard`. The net result is one `/dashboard` entry replacing the `/login` entry. The user cannot press back to return to `/login`.

</details>

---

### Q8

```jsx
function ProductPage() {
  const navigate = useNavigate();

  useEffect(() => {
    const timer = setTimeout(() => {
      navigate("/home");
    }, 3000);

    return () => clearTimeout(timer);
  }, [navigate]);

  return <p>Redirecting in 3 seconds...</p>;
}
```

#### ❓ If the user manually navigates away before 3 seconds, does `navigate("/home")` still run?

<details>
<summary>✅ Answer</summary>

```txt
No — the cleanup function (clearTimeout) prevents it.
```

**Explanation:** When the component unmounts (because the user navigated away), React runs the cleanup function returned from `useEffect`. This calls `clearTimeout(timer)`, which cancels the pending timeout. Therefore `navigate("/home")` never fires. If there were no cleanup, the timeout would still fire after 3 seconds even though the component is unmounted, causing a navigation on an unmounted component — a common bug.

</details>

---

### Q9

```jsx
// History stack: ["/home", "/about", "/contact"]  (current: "/contact")

function Component() {
  const navigate = useNavigate();

  return (
    <div>
      <button onClick={() => navigate(-1)}>Back</button>
      <button onClick={() => navigate(-3)}>Way Back</button>
    </div>
  );
}
```

#### ❓ What happens when "Back" is clicked? What happens when "Way Back" is clicked?

<details>
<summary>✅ Answer</summary>

```txt
"Back" → navigates to /about (one step back)
"Way Back" → navigates the browser to the page before the app opened (exits the app)
```

**Explanation:** `navigate(-1)` is equivalent to clicking the browser back button — it goes one entry back in the stack to `/about`. `navigate(-3)` attempts to go back 3 entries. There are only 3 entries in the stack (indices 0, 1, 2). Going back 3 from index 2 would land at index -1, which is outside the app's history — the browser navigates to whatever was open before the user entered this app. This is a real footgun.

</details>

---

### Q10

```jsx
function RedirectOnMount() {
  const navigate = useNavigate();

  navigate("/new-path");  // called directly in render body

  return null;
}
```

#### ❓ What happens when this component renders?

<details>
<summary>✅ Answer</summary>

```txt
React warning: "You should call navigate() in a React.useEffect(), not when your component is first rendering."
Possible infinite loop or unexpected behavior.
```

**Explanation:** Calling `navigate()` directly in the render body is a side effect triggered during the render phase. React may call the render function multiple times (especially in Strict Mode). The correct approach is either to use the `<Navigate>` component declaratively (`return <Navigate to="/new-path" replace />`) or to call `navigate()` inside a `useEffect`. The render body should be free of side effects.

</details>

---

## 3. Params and Search

### Q11

```jsx
// Route: <Route path="/posts/:postId/comments/:commentId" element={<Comment />} />
// URL: /posts/abc/comments/xyz

function Comment() {
  const params = useParams();
  console.log(params);
  return null;
}
```

#### ❓ What does the console log?

<details>
<summary>✅ Answer</summary>

```txt
{ postId: "abc", commentId: "xyz" }
```

**Explanation:** `useParams()` returns an object where the keys match the dynamic segment names defined in the route path (`:postId`, `:commentId`) and the values are the corresponding URL segments. All values are strings. The object only contains the params defined in the currently matched route — not inherited params from parent routes unless you are in a nested route context.

</details>

---

### Q12

```jsx
// Route: <Route path="/users/:id" element={<UserDetail />} />
// URL: /users/42

function UserDetail() {
  const { id } = useParams();
  const user = users.find(u => u.id === id);

  return <p>{user ? user.name : "Not found"}</p>;
}

const users = [{ id: 1, name: "Alice" }, { id: 42, name: "Bob" }];
```

#### ❓ What renders? Why?

<details>
<summary>✅ Answer</summary>

```txt
Not found
```

**Explanation:** URL params are always strings. `id` is the string `"42"`, but `u.id` in the array is the number `42`. The strict equality check `"42" === 42` is `false` because they are different types. The `find` returns `undefined`. Fix: `users.find(u => u.id === Number(id))` or `u.id === +id`.

</details>

---

### Q13

```jsx
// URL: /search?q=react+hooks&page=2

function SearchPage() {
  const [searchParams] = useSearchParams();

  console.log(searchParams.get("q"));
  console.log(searchParams.get("page"));
  console.log(searchParams.get("missing"));
  console.log(typeof searchParams.get("page"));
}
```

#### ❓ What are the four console outputs?

<details>
<summary>✅ Answer</summary>

```txt
"react hooks"
"2"
null
"string"
```

**Explanation:** `URLSearchParams.get()` URL-decodes the value — `+` in a query string represents a space, so `"react+hooks"` becomes `"react hooks"`. The `page` param is `"2"` as a string (all query string values are strings). A missing key returns `null`, not `undefined`. Since all values are strings, `typeof "2"` is `"string"` — convert with `Number()` when needed.

</details>

---

### Q14

```jsx
function FilterPage() {
  const [searchParams, setSearchParams] = useSearchParams();

  const handleClick = () => {
    setSearchParams({ category: "books" });
  };

  // Current URL: /filter?category=electronics&sort=price&page=3
  return <button onClick={handleClick}>Set Books</button>;
}
```

#### ❓ What is the URL after the button is clicked?

<details>
<summary>✅ Answer</summary>

```txt
/filter?category=books
```

**Explanation:** Calling `setSearchParams({ category: "books" })` with a plain object replaces the entire query string with just the provided object. All previous params (`sort`, `page`) are lost. To preserve existing params, use the functional form: `setSearchParams(prev => { prev.set("category", "books"); return prev; })`.

</details>

---

### Q15

```jsx
// URL: /items?tag=react&tag=hooks&tag=router

function TagFilter() {
  const [searchParams] = useSearchParams();

  console.log(searchParams.get("tag"));
  console.log(searchParams.getAll("tag"));
}
```

#### ❓ What are the two console outputs?

<details>
<summary>✅ Answer</summary>

```txt
"react"
["react", "hooks", "router"]
```

**Explanation:** When a query string key appears multiple times, `get()` returns only the first value. `getAll()` returns an array of all values for that key. This is the standard URLSearchParams behavior. If you need to support multi-value filters (checkboxes, multi-select), always use `getAll()` for those keys.

</details>

---

## 4. Nested Routes

### Q16

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/settings" element={<Settings />}>
          <Route index element={<GeneralSettings />} />
          <Route path="profile" element={<ProfileSettings />} />
          <Route path="security" element={<SecuritySettings />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

function Settings() {
  return (
    <div>
      <h1>Settings</h1>
      <Outlet />
    </div>
  );
}
```

#### ❓ What renders at `/settings`? What renders at `/settings/profile`?

<details>
<summary>✅ Answer</summary>

```txt
/settings         → Settings heading + GeneralSettings
/settings/profile → Settings heading + ProfileSettings
```

**Explanation:** The `index` route renders when the parent path matches exactly and no child path matches. At `/settings`, the `index` route fires and `GeneralSettings` renders inside the `<Outlet />`. At `/settings/profile`, the `profile` child route matches and `ProfileSettings` renders inside the `<Outlet />` instead. The `<h1>Settings</h1>` is always visible because it is part of the parent `Settings` layout.

</details>

---

### Q17

```jsx
function Dashboard() {
  return (
    <div>
      <Link to="reports">Reports</Link>
      <Link to="/dashboard/reports">Reports Absolute</Link>
      <Outlet />
    </div>
  );
}

// Route: <Route path="/dashboard" element={<Dashboard />}>
//          <Route path="reports" element={<Reports />} />
//        </Route>
```

#### ❓ Both links go to Reports. What is the difference between them?

<details>
<summary>✅ Answer</summary>

```txt
Both navigate to /dashboard/reports, but they work differently.

"reports" (relative) — resolved relative to the current route (/dashboard),
becomes /dashboard/reports.

"/dashboard/reports" (absolute) — always navigates to exactly /dashboard/reports,
regardless of the current route.
```

**Explanation:** Relative links in v6 are resolved relative to the nearest parent route path. Inside a component rendered at `/dashboard`, `to="reports"` resolves to `/dashboard/reports`. If this component were reused at `/admin/dashboard`, the relative link would resolve to `/admin/dashboard/reports`, while the absolute link would still go to `/dashboard/reports`. Relative links make components more reusable and portable.

</details>

---

### Q18

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/app" element={<AppLayout />}>
          <Route path="feed" element={<Feed />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

function AppLayout() {
  return (
    <div>
      <nav>App Nav</nav>
      <Outlet />
    </div>
  );
}
```

#### ❓ What renders at `/app`?

<details>
<summary>✅ Answer</summary>

```txt
App Nav (from AppLayout)
— the Outlet renders nothing (empty)
```

**Explanation:** At `/app`, the parent route matches but no child route matches. There is no `index` route defined. The `<Outlet />` renders `null`. The `AppLayout` still renders (header, nav, etc.) but the content area is empty. To show something at `/app`, add `<Route index element={<AppHome />} />` as a child.

</details>

---

### Q19

```jsx
function ProfileLayout() {
  const { username } = useOutletContext();
  return <p>User: {username}</p>;
}

function App() {
  return (
    <Routes>
      <Route
        path="/profile"
        element={<Outlet context={{ username: "alice" }} />}
      >
        <Route index element={<ProfileLayout />} />
      </Route>
    </Routes>
  );
}
```

#### ❓ What renders at `/profile`?

<details>
<summary>✅ Answer</summary>

```txt
User: alice
```

**Explanation:** You can pass a `context` prop directly on the `<Outlet>` component. Any descendant can retrieve it with `useOutletContext()`. Here, the parent route renders `<Outlet context={{ username: "alice" }} />`, and the index route's `ProfileLayout` reads `username` via `useOutletContext()`. This is a clean way to share data from a parent route down to child routes without prop drilling or context.

</details>

---

### Q20

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Root />}>
          <Route index element={<Home />} />
          <Route path="about" element={<About />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

function Root() {
  return (
    <div>
      <Outlet />
    </div>
  );
}
```

#### ❓ What renders at `/`? What renders at `/about`? What renders at `/unknown`?

<details>
<summary>✅ Answer</summary>

```txt
/        → Root wrapper + Home (index route matches)
/about   → Root wrapper + About
/unknown → nothing (no matching route, no wildcard)
```

**Explanation:** The `index` route renders inside the parent's `<Outlet />` when the parent path matches exactly. At `/`, the `Root` component renders and the `<Outlet />` is filled by `<Home />`. At `/about`, `<About />` fills the `<Outlet />`. At `/unknown`, no route matches at all — neither the parent `"/"` (because it's exact) nor any child — so nothing renders. Adding `<Route path="*" element={<NotFound />} />` would fix this.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function ProtectedRoute({ children }) {
  const { user } = useAuth();
  const location = useLocation();

  if (!user) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return children;
}
```

#### ❓ Why is `replace` important here? What happens without it?

<details>
<summary>✅ Answer</summary>

```txt
With replace: the protected route URL is removed from history — after login,
pressing back does not return the user to an infinite redirect loop.

Without replace: the protected route URL stays in history. After login,
navigating back would return to the protected page and trigger another
redirect to login, creating an endless back-button loop.
```

**Explanation:** When a user visits `/dashboard` without being logged in, they get redirected to `/login`. Without `replace`, the history stack becomes: `[..., "/dashboard", "/login"]`. After logging in and navigating to `/dashboard`, if the user presses back, they land on `/login` (which would redirect them forward again). With `replace`, `"/dashboard"` is replaced by `"/login"` in the stack: `[..., "/login"]`. After login they go to `/dashboard`, and pressing back takes them somewhere sensible.

</details>

---

### Q22

```jsx
// Page A navigates to Page B with state
navigate("/page-b", { state: { message: "Hello from A" } });

// Page B reads the state
function PageB() {
  const location = useLocation();
  console.log(location.state);
  return null;
}
```

#### ❓ What happens to the state if the user refreshes the browser while on Page B?

<details>
<summary>✅ Answer</summary>

```txt
The state is preserved after a refresh.
console.log(location.state) → { message: "Hello from A" }
```

**Explanation:** React Router stores location state in the browser's session history via the `history.pushState` / `history.replaceState` API. The browser preserves history state entries through page refreshes for the same session (until the tab is closed). However, state is NOT preserved if the user opens a new tab to the same URL or navigates directly to the URL — in those cases `location.state` is `null`. Always provide a fallback: `location.state?.message ?? "default"`.

</details>

---

### Q23

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Navigate to="/home" />} />
        <Route path="/home" element={<Home />} />
      </Routes>
    </BrowserRouter>
  );
}
```

#### ❓ What happens at `/`? Is there any problem with this approach?

<details>
<summary>✅ Answer</summary>

```txt
At "/", the Navigate component renders and immediately redirects to "/home".
The user sees Home.

Potential problem: without "replace", pressing back from /home returns to /,
which redirects to /home again — an infinite loop in navigation UX.
```

**Explanation:** `<Navigate to="/home" />` without `replace` pushes `/home` onto the history stack while `/` stays below it. When the user presses back, the browser returns to `/`, which renders `<Navigate to="/home" />` again, pushing another `/home` entry. The fix is `<Navigate to="/home" replace />` — this replaces `/` with `/home` in the history stack, so the back button goes to wherever the user was before `/`.

</details>

---

### Q24

```jsx
function UserProfile() {
  const { id } = useParams();
  const location = useLocation();

  useEffect(() => {
    fetchUserData(id);
  }, []);  // empty dependency array

  return <p>Profile for {id}</p>;
}

// Route: /users/:id
// User navigates: /users/1 → /users/2 (without unmounting)
```

#### ❓ Does `fetchUserData` run when navigating from `/users/1` to `/users/2`?

<details>
<summary>✅ Answer</summary>

```txt
No — fetchUserData does NOT re-run.
The component stays mounted, the effect has [] deps, so it only runs once on mount.
The page still shows "Profile for 2" (the id in JSX updates) but data is stale.
```

**Explanation:** In React Router v6, navigating between the same route pattern with different params does not unmount and remount the component. The `id` param updates (so the JSX reflects the new ID), but effects with `[]` only run on mount. The data fetched in the effect still belongs to user 1. Fix: add `id` to the effect dependency array: `useEffect(() => { fetchUserData(id); }, [id])`.

</details>

---

### Q25

```jsx
function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/users" element={<UserList />} />
        <Route path="/users/:id" element={<UserDetail />} />
      </Routes>
    </BrowserRouter>
  );
}

function UserList() {
  return (
    <div>
      <Link to="new">Create New User</Link>
    </div>
  );
}
```

#### ❓ Where does the "Create New User" link navigate to?

<details>
<summary>✅ Answer</summary>

```txt
/users/new
```

**Explanation:** `UserList` is rendered at the route `/users`. A relative `to="new"` link is resolved relative to the current route's path. Since the current route is `/users`, `to="new"` resolves to `/users/new`. Note that `/users/new` matches the `:id` dynamic segment route — it would render `UserDetail` with `id = "new"`. If you want a separate static route, add `<Route path="/users/new" element={<NewUser />} />` — v6's specificity ranking will ensure `new` takes precedence over `:id`.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Route Matching | Q1–Q5 | Specificity ranking, exact matching, wildcard, missing Outlet |
| Navigation | Q6–Q10 | push vs replace, navigate(-n), cleanup, render-phase navigate |
| Params and Search | Q11–Q15 | String coercion, get vs getAll, setSearchParams object vs functional |
| Nested Routes | Q16–Q20 | Index routes, relative vs absolute links, Outlet context, empty Outlet |
| Edge Cases | Q21–Q25 | replace on redirect, location state on refresh, Navigate loop, stale effect, relative Link from list |
