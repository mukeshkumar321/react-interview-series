# React useContext Hook

## Table of Contents

1. [The Problem Context Solves](#1-the-problem-context-solves)
2. [What is React Context](#2-what-is-react-context)
3. [Context API — Three Steps](#3-context-api--three-steps)
4. [useContext Syntax](#4-usecontext-syntax)
5. [Context Default Value](#5-context-default-value)
6. [Context Re-renders](#6-context-re-renders)
7. [Context Performance Optimization](#7-context-performance-optimization)
8. [Multiple Contexts](#8-multiple-contexts)
9. [Context vs Props vs State Management](#9-context-vs-props-vs-state-management)
10. [Common Context Patterns](#10-common-context-patterns)
11. [Context with useReducer](#11-context-with-usereducer)
12. [Context and Testing](#12-context-and-testing)
13. [Common Mistakes](#13-common-mistakes)
14. [Best Practices](#14-best-practices)

---

## 1. The Problem Context Solves

### Props Drilling

Props drilling is the practice of passing data through multiple layers of components that do not need the data — solely to deliver it to a deeply nested component that does.

### Visual Diagram

```text
App (holds: user = { name: 'Alice', role: 'admin' })
  ↓ passes user as prop
  Layout
    ↓ passes user as prop (Layout doesn't use it)
    Sidebar
      ↓ passes user as prop (Sidebar doesn't use it)
      Navigation
        ↓ passes user as prop (Navigation doesn't use it)
        UserAvatar ← only this component actually uses user
```

### Code Illustration of the Problem

```jsx
// ❌ Props drilling — user is passed through 4 layers
function App() {
  const [user] = useState({ name: 'Alice', role: 'admin' });
  return <Layout user={user} />;
}

function Layout({ user }) {      // doesn't use user — just passes it down
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {     // doesn't use user — just passes it down
  return <Navigation user={user} />;
}

function Navigation({ user }) {  // doesn't use user — just passes it down
  return <UserAvatar user={user} />;
}

function UserAvatar({ user }) {  // finally uses user
  return <img src={user.avatar} alt={user.name} />;
}
```

### Why This Is Painful

| Problem | Consequence |
|---------|------------|
| Intermediate components receive props they don't need | Cluttered API, harder to understand |
| Renaming or restructuring data requires updating every layer | High maintenance cost |
| Adding a new consumer deep in the tree requires threading props through every ancestor | Brittle architecture |
| Intermediate components re-render when user changes even if they don't use it | Performance degradation |

---

## 2. What is React Context

React Context is a mechanism for making data accessible to any component in the tree **without passing it through props manually at every level**.

### Key Concepts

| Concept | Description |
|---------|------------|
| `createContext` | Creates a context object with an optional default value |
| `Provider` | A component that wraps part of the tree and supplies a value |
| `useContext` | A hook that reads the current value from a context |
| `Consumer` | The legacy render-prop alternative to `useContext` (not recommended) |

### Important Clarification

Context is **not a state management solution**. It is a **transport mechanism** — a way to deliver data from a provider to consumers without manual prop threading.

```text
Context does:
  ✅ Make data accessible anywhere in the subtree
  ✅ Avoid props drilling
  ✅ Work with local state, useReducer, or any external store

Context does NOT:
  ❌ Manage or store state by itself (that is state, not context)
  ❌ Replace Redux, Zustand, or Jotai for complex state logic
  ❌ Optimize re-renders automatically
```

### Mental Model

```text
Without Context:
  Data flows → → → → → through every intermediate component

With Context:
  Provider wraps subtree
    ↓
  Data is "broadcast" to the subtree
    ↓
  Any consumer anywhere in the subtree reads the data directly
  (intermediaries are bypassed entirely)
```

---

## 3. Context API — Three Steps

### Step 1 — Create the Context

```jsx
import { createContext } from 'react';

// createContext(defaultValue)
// defaultValue is used when a component consumes context outside a Provider
const UserContext = createContext(null);

export default UserContext;
```

### Step 2 — Wrap the Tree with Provider and Supply a Value

```jsx
import { useState } from 'react';
import UserContext from './UserContext';

function App() {
  const [user, setUser] = useState({ name: 'Alice', role: 'admin' });

  return (
    // Provider wraps the tree; value prop is what consumers will receive
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}
```

### Step 3 — Consume with useContext

```jsx
import { useContext } from 'react';
import UserContext from './UserContext';

// Deep inside the tree — no props needed
function UserAvatar() {
  const user = useContext(UserContext);
  return <img src={user.avatar} alt={user.name} />;
}
```

### Full Working Example — All Three Steps Together

```jsx
import { createContext, useContext, useState } from 'react';

// Step 1: Create
const ThemeContext = createContext('light');

// Step 2: Provide
function App() {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={theme}>
      <Header />
      <main>
        <Page />
      </main>
      <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
        Toggle Theme
      </button>
    </ThemeContext.Provider>
  );
}

// Intermediate — does not receive theme as prop
function Header() {
  return <nav><ThemeIndicator /></nav>;
}

// Step 3: Consume
function ThemeIndicator() {
  const theme = useContext(ThemeContext);
  return <span>Current theme: {theme}</span>;
}

// Also a consumer — any component in the subtree can consume
function Page() {
  const theme = useContext(ThemeContext);
  return <div className={`page page--${theme}`}>Content</div>;
}
```

---

## 4. useContext Syntax

### Basic Usage

```jsx
const value = useContext(MyContext);
```

- Takes the context object (the return value of `createContext`) as its argument
- Returns the current value from the nearest matching `Provider` above it in the tree
- If no `Provider` is found, returns the default value passed to `createContext`

### Rules

```text
✅ Must be called at the top level of a function component or custom hook
✅ Must be called inside a function component — not in loops, conditions, or event handlers
✅ The context object passed to useContext must be the exact object from createContext
✅ Works anywhere in the component tree — no intermediate components needed
```

### What useContext Returns

```jsx
// The value returned is exactly what the nearest Provider's value prop holds
const ThemeContext = createContext('light');

function Provider({ children }) {
  return (
    <ThemeContext.Provider value="dark">
      {children}
    </ThemeContext.Provider>
  );
}

function Consumer() {
  const theme = useContext(ThemeContext);
  console.log(theme); // "dark" — from the nearest Provider
}
```

### Comparison: useContext vs Consumer (Legacy)

```jsx
// ❌ Legacy Consumer pattern (render props — harder to read)
function OldWay() {
  return (
    <ThemeContext.Consumer>
      {(theme) => (
        <div className={theme}>Content</div>
      )}
    </ThemeContext.Consumer>
  );
}

// ✅ Modern useContext (cleaner, composable)
function NewWay() {
  const theme = useContext(ThemeContext);
  return <div className={theme}>Content</div>;
}
```

---

## 5. Context Default Value

### What is the Default Value

The argument passed to `createContext(defaultValue)` is used when a component calls `useContext` but is **not inside any matching Provider**. It is not the initial value for the Provider — it is a fallback for consumers with no Provider ancestor.

```jsx
const CountContext = createContext(0); // default value is 0

// Consumer inside a Provider
function InsideProvider() {
  const count = useContext(CountContext);
  console.log(count); // whatever value the Provider supplies
}

// Consumer OUTSIDE any Provider
function OutsideProvider() {
  const count = useContext(CountContext);
  console.log(count); // 0 — the default value
}

function App() {
  return (
    <div>
      <CountContext.Provider value={42}>
        <InsideProvider /> {/* logs 42 */}
      </CountContext.Provider>
      <OutsideProvider /> {/* logs 0 — no Provider above */}
    </div>
  );
}
```

### Common Default Value Patterns

```jsx
// Pattern 1: null default (explicit — forces consumers to check)
const AuthContext = createContext(null);

// Pattern 2: shape-matching default (useful for TypeScript)
const AuthContext = createContext({
  user: null,
  login: () => {},
  logout: () => {},
});

// Pattern 3: throw on missing Provider (defensive programming)
const StrictContext = createContext(null);

function useStrictContext() {
  const value = useContext(StrictContext);
  if (value === null) {
    throw new Error('useStrictContext must be used within StrictContext.Provider');
  }
  return value;
}
```

### Provider value={undefined} vs No Provider

```jsx
const MyContext = createContext('DEFAULT');

// No Provider at all → default value 'DEFAULT' is used
function NoProvider() {
  const v = useContext(MyContext);
  console.log(v); // 'DEFAULT'
}

// Provider present but value={undefined} → undefined is used, NOT the default
function WithUndefined() {
  return (
    <MyContext.Provider value={undefined}>
      <Consumer />
    </MyContext.Provider>
  );
}

function Consumer() {
  const v = useContext(MyContext);
  console.log(v); // undefined — Provider overrides default, even with undefined
}
```

This is a subtle but important distinction: the default value is only used when there is **no Provider** above. Providing `undefined` explicitly overrides the default.

---

## 6. Context Re-renders

### How Context Triggers Re-renders

When a context `Provider`'s `value` prop changes, React re-renders **all components** that call `useContext` with that context — regardless of which part of the value they actually use.

```jsx
const AppContext = createContext({ count: 0, user: null });

function App() {
  const [count, setCount] = useState(0);
  const [user, setUser] = useState({ name: 'Alice' });

  return (
    <AppContext.Provider value={{ count, user }}>
      <CountDisplay />   {/* uses count */}
      <UserDisplay />    {/* uses user */}
    </AppContext.Provider>
  );
}

function CountDisplay() {
  const { count } = useContext(AppContext);
  console.log('CountDisplay rendered'); // logs on EVERY value change — even user changes
  return <p>{count}</p>;
}

function UserDisplay() {
  const { user } = useContext(AppContext);
  console.log('UserDisplay rendered'); // logs on EVERY value change — even count changes
  return <p>{user.name}</p>;
}
```

### Why All Consumers Re-render

React's context comparison uses `Object.is` on the entire `value` prop. It does not compare individual fields within the value. When `value` is a new object reference (as in `{ count, user }` above), React considers the context changed and notifies all consumers.

```text
setCount(1) called
  ↓
App re-renders
  ↓
value = { count: 1, user: {...} } ← new object (different reference)
  ↓
Object.is(prevValue, newValue) → false
  ↓
All consumers notified: CountDisplay re-renders, UserDisplay re-renders
  (even though user did not change)
```

### Re-render Trigger Conditions

| Scenario | Consumers re-render? |
|----------|---------------------|
| `value` reference changes (new object) | ✅ Yes — all consumers |
| `value` reference is the same (`Object.is` true) | ❌ No |
| Consumer's own state or props change | ✅ Yes (normal re-render, unrelated to context) |
| Ancestor of Provider re-renders but value doesn't change | ❌ No (if value reference is stable) |

---

## 7. Context Performance Optimization

### Problem: Object Literal as Context Value

The most common performance issue with context is creating a new object literal as the `value` prop on every render.

```jsx
// ❌ New object every render → all consumers re-render every render
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  return (
    <AppContext.Provider value={{ user, theme, setUser, setTheme }}>
      {/* Every render: value = new {} → all consumers re-render */}
      <Children />
    </AppContext.Provider>
  );
}
```

### Fix 1 — Memoize the Context Value with useMemo

```jsx
// ✅ Value only changes when user or theme actually changes
function App() {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  const contextValue = useMemo(() => ({
    user,
    theme,
    setUser,
    setTheme,
  }), [user, theme]); // setUser and setTheme are stable — no need in deps

  return (
    <AppContext.Provider value={contextValue}>
      <Children />
    </AppContext.Provider>
  );
}
```

### Fix 2 — Split Contexts

Instead of one large context, split data into separate contexts by domain. Consumers only subscribe to the context they need.

```jsx
// ❌ One large context — all consumers re-render when anything changes
const AppContext = createContext({ user: null, theme: 'light', cart: [] });

// ✅ Split contexts — consumers only re-render for their specific data
const UserContext  = createContext(null);
const ThemeContext = createContext('light');
const CartContext  = createContext([]);

function App() {
  const [user, setUser]   = useState(null);
  const [theme, setTheme] = useState('light');
  const [cart, setCart]   = useState([]);

  return (
    <UserContext.Provider value={user}>
      <ThemeContext.Provider value={theme}>
        <CartContext.Provider value={cart}>
          <Children />
        </CartContext.Provider>
      </ThemeContext.Provider>
    </UserContext.Provider>
  );
}

// This component only re-renders when theme changes — not when user or cart changes
function ThemeToggle() {
  const theme = useContext(ThemeContext);
  return <button>{theme}</button>;
}
```

### Fix 3 — Separate State and Dispatch Contexts

When using `useReducer`, putting state and dispatch in separate contexts means components that only need to dispatch actions won't re-render when state changes.

```jsx
const StateContext    = createContext(null);
const DispatchContext = createContext(null);

function Provider({ children }) {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <DispatchContext.Provider value={dispatch}>
      {/* dispatch is stable — DispatchContext never changes */}
      <StateContext.Provider value={state}>
        {children}
      </StateContext.Provider>
    </DispatchContext.Provider>
  );
}

// Only re-renders when state changes
function DataDisplay() {
  const state = useContext(StateContext);
  return <p>{state.count}</p>;
}

// Never re-renders due to context (dispatch is stable)
function ActionButton() {
  const dispatch = useContext(DispatchContext);
  return <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>;
}
```

---

## 8. Multiple Contexts

### Stacking Providers

React supports any number of Providers. Components consume whichever contexts they need.

```jsx
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext('light');
const AuthContext  = createContext(null);
const LangContext  = createContext('en');

function App() {
  const [theme, setTheme]   = useState('light');
  const [user, setUser]     = useState(null);
  const [lang, setLang]     = useState('en');

  return (
    <ThemeContext.Provider value={theme}>
      <AuthContext.Provider value={{ user, setUser }}>
        <LangContext.Provider value={lang}>
          <Dashboard />
        </LangContext.Provider>
      </AuthContext.Provider>
    </ThemeContext.Provider>
  );
}

// Component using multiple contexts
function UserGreeting() {
  const theme = useContext(ThemeContext);
  const { user } = useContext(AuthContext);
  const lang = useContext(LangContext);

  const greeting = lang === 'en' ? 'Hello' : 'Hola';
  return (
    <div className={`greeting greeting--${theme}`}>
      {greeting}, {user?.name}
    </div>
  );
}
```

### Context Composition Pattern

For cleanliness, combine all providers into a single `AppProviders` component:

```jsx
function AppProviders({ children }) {
  return (
    <ThemeProvider>
      <AuthProvider>
        <CartProvider>
          <NotificationProvider>
            {children}
          </NotificationProvider>
        </CartProvider>
      </AuthProvider>
    </ThemeProvider>
  );
}

function App() {
  return (
    <AppProviders>
      <Router>
        <Routes />
      </Router>
    </AppProviders>
  );
}
```

### When to Combine vs Split Contexts

| Combine into one context | Split into separate contexts |
|--------------------------|------------------------------|
| Data is always used together | Data is used independently |
| Updates to parts are rare | Parts update at different frequencies |
| Only a few consumers | Many consumers with different needs |
| Simplicity is the priority | Performance is the priority |

---

## 9. Context vs Props vs State Management

### Decision Framework

```text
Is the data used in only one component or a few tightly coupled components?
  → Use local state + props

Is the data needed by many components across different parts of the tree,
and it does not change very frequently?
  → Use Context

Is the data a high-frequency global state (user interactions, real-time data)
or requires complex update logic (optimistic updates, middleware, devtools)?
  → Use a dedicated state management library (Redux, Zustand, Jotai, etc.)
```

### Comparison Table

| Dimension | Props | Context | Redux / Zustand |
|-----------|-------|---------|-----------------|
| Data scope | Parent to child | Any level in subtree | Anywhere in app |
| Update frequency | Any | Low to medium | Any |
| Boilerplate | Low | Low to medium | Medium to high |
| Performance on frequent updates | Good (co-located) | Poor (all consumers) | Good (selectors) |
| DevTools support | None needed | React DevTools | Dedicated devtools |
| Complexity | Low | Low | Medium to high |
| Best for | Co-located data | Auth, theme, i18n | Shopping cart, live data |

### Practical Examples

```jsx
// ✅ Props — correct for co-located state
function Form() {
  const [value, setValue] = useState('');
  return <Input value={value} onChange={setValue} />;
}

// ✅ Context — correct for cross-cutting infrequent data
const AuthContext = createContext(null);
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  return <AuthContext.Provider value={{ user, setUser }}>{children}</AuthContext.Provider>;
}

// ✅ Zustand — correct for frequent global state
const useCartStore = create((set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
}));
```

---

## 10. Common Context Patterns

### Pattern 1 — Auth Context

The most common use case. Stores the authenticated user and provides login/logout functions.

```jsx
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  const login = useCallback(async (credentials) => {
    const loggedInUser = await authAPI.login(credentials);
    setUser(loggedInUser);
  }, []);

  const logout = useCallback(async () => {
    await authAPI.logout();
    setUser(null);
  }, []);

  const value = useMemo(() => ({
    user,
    login,
    logout,
    isAuthenticated: user !== null,
  }), [user, login, logout]);

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook for clean consumption
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}

// Usage in any component
function NavBar() {
  const { user, logout, isAuthenticated } = useAuth();
  return (
    <nav>
      {isAuthenticated ? (
        <>
          <span>Hello, {user.name}</span>
          <button onClick={logout}>Logout</button>
        </>
      ) : (
        <a href="/login">Login</a>
      )}
    </nav>
  );
}
```

### Pattern 2 — Theme Context

```jsx
const ThemeContext = createContext({ theme: 'light', toggle: () => {} });

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');

  const toggle = useCallback(() => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  }, []);

  const value = useMemo(() => ({ theme, toggle }), [theme, toggle]);

  return (
    <ThemeContext.Provider value={value}>
      <div data-theme={theme}>{children}</div>
    </ThemeContext.Provider>
  );
}

function useTheme() {
  return useContext(ThemeContext);
}

function ThemeToggleButton() {
  const { theme, toggle } = useTheme();
  return (
    <button onClick={toggle}>
      Switch to {theme === 'light' ? 'dark' : 'light'} mode
    </button>
  );
}
```

### Pattern 3 — i18n / Language Context

```jsx
const LanguageContext = createContext({ lang: 'en', t: (key) => key });

function LanguageProvider({ children }) {
  const [lang, setLang] = useState('en');

  const translations = useMemo(() => ({
    en: { greeting: 'Hello', farewell: 'Goodbye' },
    es: { greeting: 'Hola', farewell: 'Adiós' },
  }), []);

  const t = useCallback((key) => {
    return translations[lang]?.[key] ?? key;
  }, [lang, translations]);

  const value = useMemo(() => ({ lang, setLang, t }), [lang, setLang, t]);

  return (
    <LanguageContext.Provider value={value}>
      {children}
    </LanguageContext.Provider>
  );
}
```

---

## 11. Context with useReducer

### The Pattern — Redux-Like Without Redux

Combining `useContext` and `useReducer` provides a lightweight global state pattern. The reducer handles complex state transitions; context transports the state and dispatch function.

```jsx
import { createContext, useContext, useReducer, useMemo } from 'react';

// 1. Define reducer and initial state
const initialState = { todos: [], filter: 'all' };

function todoReducer(state, action) {
  switch (action.type) {
    case 'ADD_TODO':
      return {
        ...state,
        todos: [...state.todos, { id: Date.now(), text: action.payload, done: false }],
      };
    case 'TOGGLE_TODO':
      return {
        ...state,
        todos: state.todos.map(t =>
          t.id === action.payload ? { ...t, done: !t.done } : t
        ),
      };
    case 'SET_FILTER':
      return { ...state, filter: action.payload };
    default:
      return state;
  }
}

// 2. Create contexts
const TodoStateContext    = createContext(null);
const TodoDispatchContext = createContext(null);

// 3. Provider
function TodoProvider({ children }) {
  const [state, dispatch] = useReducer(todoReducer, initialState);

  // Memoize state context value
  const stateValue = useMemo(() => state, [state]);
  // dispatch is already stable — no useMemo needed

  return (
    <TodoDispatchContext.Provider value={dispatch}>
      <TodoStateContext.Provider value={stateValue}>
        {children}
      </TodoStateContext.Provider>
    </TodoDispatchContext.Provider>
  );
}

// 4. Custom hooks for consumers
function useTodoState()    { return useContext(TodoStateContext); }
function useTodoDispatch() { return useContext(TodoDispatchContext); }

// 5. Usage in components
function TodoList() {
  const { todos, filter } = useTodoState();

  const visible = todos.filter(t => {
    if (filter === 'active') return !t.done;
    if (filter === 'done')   return t.done;
    return true;
  });

  return (
    <ul>
      {visible.map(todo => <TodoItem key={todo.id} todo={todo} />)}
    </ul>
  );
}

function TodoItem({ todo }) {
  const dispatch = useTodoDispatch(); // only needs dispatch — won't re-render on state change

  return (
    <li
      style={{ textDecoration: todo.done ? 'line-through' : 'none' }}
      onClick={() => dispatch({ type: 'TOGGLE_TODO', payload: todo.id })}
    >
      {todo.text}
    </li>
  );
}

function AddTodo() {
  const dispatch = useTodoDispatch();
  const [text, setText] = useState('');

  const handleAdd = () => {
    if (text.trim()) {
      dispatch({ type: 'ADD_TODO', payload: text });
      setText('');
    }
  };

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <button onClick={handleAdd}>Add</button>
    </div>
  );
}
```

### Why Separate State and Dispatch Contexts

```text
Combined context: { state, dispatch }
  → Every dispatch call changes state → new context value → ALL consumers re-render
  → AddTodo re-renders every time any todo is added/toggled (even if it doesn't display state)

Separate contexts: StateContext + DispatchContext
  → dispatch is stable (useReducer guarantee)
  → DispatchContext.value never changes
  → Components consuming only DispatchContext NEVER re-render due to context
  → 
```

---

## 12. Context and Testing

### Testing Components That Consume Context

Components using `useContext` must be rendered inside the appropriate Provider in tests. Otherwise they receive the default value (or throw if using the "throw on missing provider" pattern).

```jsx
// Component under test
function UserGreeting() {
  const { user } = useContext(AuthContext);
  if (!user) return <p>Not logged in</p>;
  return <p>Hello, {user.name}</p>;
}

// Test — wrap with Provider
import { render, screen } from '@testing-library/react';

test('shows greeting when logged in', () => {
  const mockUser = { name: 'Alice', role: 'admin' };

  render(
    <AuthContext.Provider value={{ user: mockUser, login: jest.fn(), logout: jest.fn() }}>
      <UserGreeting />
    </AuthContext.Provider>
  );

  expect(screen.getByText('Hello, Alice')).toBeInTheDocument();
});

test('shows fallback when not logged in', () => {
  render(
    <AuthContext.Provider value={{ user: null, login: jest.fn(), logout: jest.fn() }}>
      <UserGreeting />
    </AuthContext.Provider>
  );

  expect(screen.getByText('Not logged in')).toBeInTheDocument();
});
```

### Reusable Test Wrapper

```jsx
// test-utils.jsx
function AllProviders({ children }) {
  return (
    <AuthContext.Provider value={{ user: null, login: jest.fn(), logout: jest.fn() }}>
      <ThemeContext.Provider value={{ theme: 'light', toggle: jest.fn() }}>
        {children}
      </ThemeContext.Provider>
    </AuthContext.Provider>
  );
}

const customRender = (ui, options) =>
  render(ui, { wrapper: AllProviders, ...options });

export { customRender as render };

// Usage in tests
import { render } from './test-utils';

test('renders with providers', () => {
  render(<MyComponent />);
  // ...
});
```

---

## 13. Common Mistakes

### Mistake 1 — Using Context for High-Frequency Updates

Context is not optimized for frequent updates. Every value change re-renders all consumers. For data that changes rapidly (mouse position, scroll position, form field values), use local state or a library with subscription-based selectors.

```jsx
// ❌ Context for high-frequency state — every keystroke re-renders all consumers
const FormContext = createContext({});

function FormProvider({ children }) {
  const [values, setValues] = useState({ name: '', email: '', phone: '' });

  return (
    <FormContext.Provider value={{ values, setValues }}>
      {children}
    </FormContext.Provider>
  );
}
// Every keypress → setValues → new context value → all consumers re-render

// ✅ Use local state or a form library (React Hook Form) instead
function Form() {
  const [name, setName]   = useState('');
  const [email, setEmail] = useState('');
  // Each input manages its own state — no cross-component re-renders
  return <form>...</form>;
}
```

### Mistake 2 — Not Memoizing the Context Value

When the Provider is inside a component, its `value` prop creates a new object on every render unless memoized.

```jsx
// ❌ New object every render → all consumers re-render on every App render
function App() {
  const [user, setUser] = useState(null);

  return (
    <AuthContext.Provider value={{ user, setUser }}> {/* new {} every render */}
      <Router />
    </AuthContext.Provider>
  );
}

// ✅ Memoized value — only changes when user changes
function App() {
  const [user, setUser] = useState(null);

  const authValue = useMemo(() => ({ user, setUser }), [user]);
  // setUser is stable — not needed in deps

  return (
    <AuthContext.Provider value={authValue}>
      <Router />
    </AuthContext.Provider>
  );
}
```

### Mistake 3 — One Giant Context with Everything

Putting all app data in a single context forces all consumers to re-render whenever anything changes.

```jsx
// ❌ Single mega-context
const AppContext = createContext({
  user: null,
  theme: 'light',
  cart: [],
  notifications: [],
  language: 'en',
});
// Updating notifications forces user-related components to re-render

// ✅ Split by domain
const AuthContext         = createContext(null);
const ThemeContext         = createContext('light');
const CartContext          = createContext([]);
const NotificationContext  = createContext([]);
const LanguageContext      = createContext('en');
```

### Mistake 4 — Forgetting the value Prop on Provider

If you render a Provider without a `value` prop, the consumers receive `undefined`, not the default value.

```jsx
// ❌ No value prop — consumers get undefined
<UserContext.Provider>
  <App />
</UserContext.Provider>

// ✅ Always provide value
<UserContext.Provider value={user}>
  <App />
</UserContext.Provider>
```

### Mistake 5 — Using Context Instead of Component Composition

Sometimes props drilling can be avoided entirely through better component composition, without introducing context at all.

```jsx
// ❌ Context added to avoid prop drilling — but composition works here
function App() {
  const [user] = useState({ name: 'Alice' });
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

function Layout() {
  return <Sidebar />;
}

function Sidebar() {
  return <UserPanel />;
}

function UserPanel() {
  const user = useContext(UserContext);
  return <p>{user.name}</p>;
}

// ✅ Composition — pass JSX directly, no context needed
function App() {
  const [user] = useState({ name: 'Alice' });
  return (
    <Layout sidebar={<UserPanel user={user} />} />
  );
}

function Layout({ sidebar }) {
  return <div>{sidebar}</div>; // receives and renders JSX — no user knowledge needed
}
```

---

## 14. Best Practices

### 1 — Split Contexts by Domain and Update Frequency

Keep each context focused on a single concern. Split contexts that update independently so consumers are not over-notified.

```jsx
// ✅ One context per domain
const AuthContext    = createContext(null);  // updates on login/logout
const ThemeContext   = createContext(null);  // updates on theme toggle
const CartContext    = createContext(null);  // updates on cart changes
```

### 2 — Always Memoize the Context Value

Wrap the context value in `useMemo` whenever the Provider lives inside a component that can re-render.

```jsx
// ✅ Stable value — only re-triggers consumers when data changes
const value = useMemo(() => ({ user, login, logout }), [user, login, logout]);
return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
```

### 3 — Create Custom Hooks for Context Consumption

Exposing context through a custom hook centralizes error handling, improves readability, and hides implementation details from consumers.

```jsx
// ✅ Custom hook pattern
function useAuth() {
  const context = useContext(AuthContext);
  if (context === null) {
    throw new Error('useAuth must be used within <AuthProvider>');
  }
  return context;
}

// Consumer — clean, no import of AuthContext needed
function Profile() {
  const { user } = useAuth(); // clear intent
  return <p>{user.name}</p>;
}
```

### 4 — Keep Context Minimal — Only Truly Shared Data

Ask: "Does this data need to be accessible from multiple, unrelated parts of the app?" If no, keep it as local state or props.

```text
Good candidates for context:
  ✅ Authenticated user (needed everywhere: nav, sidebar, pages)
  ✅ Theme (affects every UI component)
  ✅ Language/locale (affects all text)

Poor candidates for context:
  ❌ Form field values (local to the form)
  ❌ Hover state (local to a specific component)
  ❌ Modal open/closed state (usually local)
  ❌ Pagination state (specific to a list component)
```

### 5 — Co-locate Context Near Where It Is Used

Do not always put Providers at the top of the app. Place them as low in the tree as possible — only above the components that need the data.

```jsx
// ❌ Auth at root — fine. But this modal context doesn't need to be at root.
function App() {
  return (
    <AuthProvider>
      <ModalProvider>  {/* ← only needed in the dashboard */}
        <Router>...</Router>
      </ModalProvider>
    </AuthProvider>
  );
}

// ✅ ModalProvider co-located where it is actually needed
function Dashboard() {
  return (
    <ModalProvider>
      <DashboardLayout />
    </ModalProvider>
  );
}
```

### 6 — Summary Table

| Practice | Why |
|----------|-----|
| Split contexts by domain | Consumers only re-render when their specific data changes |
| `useMemo` on context value | Prevents unnecessary re-renders from new object references |
| Custom hook with error check | Provides better DX and catches missing Provider errors early |
| Keep context minimal | Reduces blast radius of re-renders |
| Co-locate Providers low in the tree | Limits which components are affected by updates |
| Avoid context for high-frequency updates | Context re-renders all consumers — use local state or a library |
| Separate state and dispatch contexts | Dispatch consumers never re-render when state changes |

---
