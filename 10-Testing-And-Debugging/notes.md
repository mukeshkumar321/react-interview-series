# React Testing and Debugging

## Table of Contents

1. [Why Testing Matters in React](#1-why-testing-matters-in-react)
2. [Testing Stack](#2-testing-stack)
3. [React Testing Library Philosophy](#3-react-testing-library-philosophy)
4. [Basic Test Structure](#4-basic-test-structure)
5. [Queries](#5-queries)
6. [User Interactions](#6-user-interactions)
7. [Async Testing](#7-async-testing)
8. [Mocking](#8-mocking)
9. [Testing Custom Hooks](#9-testing-custom-hooks)
10. [Component Testing Patterns](#10-component-testing-patterns)
11. [Testing with Context](#11-testing-with-context)
12. [Snapshot Testing](#12-snapshot-testing)
13. [Debugging Techniques](#13-debugging-techniques)
14. [Common React Bugs](#14-common-react-bugs)
15. [Error Boundaries for Debugging](#15-error-boundaries-for-debugging)
16. [Using React DevTools Profiler](#16-using-react-devtools-profiler)
17. [Common Testing Mistakes](#17-common-testing-mistakes)
18. [Best Practices](#18-best-practices)

---

## 1. Why Testing Matters in React

Testing is not optional in professional React development. It is the mechanism that converts a codebase from something fragile into something that can be confidently maintained, refactored, and extended.

### Catch Bugs Before Users See Them

Every untested code path is a potential bug waiting to surface in production. Automated tests create a safety net that runs on every commit, blocking regressions before they reach users.

```text
Developer writes code
        ↓
Tests run automatically (CI/CD pipeline)
        ↓
Failing tests block the merge
        ↓
Only verified code reaches production
```

### Confidence in Refactoring

Without tests, changing internal implementation is a gamble. With a solid test suite, you can aggressively refactor — rename variables, extract components, switch state management libraries — while the tests verify that observable behavior remains unchanged.

### Documentation Through Tests

A test file for a component is simultaneously runnable documentation. It shows:
- What props the component accepts and how it responds to them
- What happens when a user clicks a button
- What renders under specific conditions
- What error states look like

A new team member reading `LoginForm.test.jsx` immediately understands every supported scenario of that component.

### Types of Tests

| Type | Scope | Speed | Confidence | Tool |
|---|---|---|---|---|
| Unit | Single function / component | Very fast | Low per test | Jest, Vitest |
| Integration | Multiple components together | Fast | Medium-high | RTL + Jest |
| End-to-End | Full user journey in browser | Slow | High | Cypress, Playwright |
| Snapshot | Component rendered output | Fast | Low–Medium | Jest |
| Accessibility | ARIA and role violations | Fast | Medium | jest-axe |

### The Testing Trophy (Kent C. Dodds)

The "testing trophy" model recommends investing most in integration tests:

```text
         /\
        /  \    E2E tests (few, expensive, high confidence)
       /----\
      /      \  Integration tests (most of your tests)
     /--------\
    /          \ Unit tests (focused utilities, hooks)
   /____________\
      Static analysis (TypeScript, ESLint)
```

Most value per dollar in a React codebase comes from integration tests — tests that render real components with real interactions.

---

## 2. Testing Stack

### Jest

Jest is a JavaScript test runner created by Meta. It provides:
- Test runner (`jest` CLI)
- Assertion library (`expect`)
- Mocking utilities (`jest.fn()`, `jest.mock()`, `jest.spyOn()`)
- Code coverage reports (`--coverage` flag)
- Snapshot testing

```bash
npm install --save-dev jest @types/jest babel-jest @babel/preset-env @babel/preset-react
```

```js
// jest.config.js
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterFramework: ['@testing-library/jest-dom'],
  transform: {
    '^.+\\.(js|jsx|ts|tsx)$': 'babel-jest',
  },
  moduleNameMapper: {
    '\\.(css|less|scss)$': 'identity-obj-proxy',
    '\\.(png|jpg|gif|svg)$': '<rootDir>/__mocks__/fileMock.js',
  },
};
```

### React Testing Library (RTL)

RTL renders components into a real DOM (via jsdom) and provides utilities to query and interact with that DOM. Its philosophy centers on testing from the user's perspective rather than the implementation's perspective.

```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom @testing-library/user-event
```

RTL extends Jest's `expect` with custom matchers from `@testing-library/jest-dom`:

```js
expect(element).toBeInTheDocument();
expect(element).toHaveTextContent('Hello');
expect(element).toBeDisabled();
expect(element).toHaveValue('input value');
expect(element).toBeVisible();
expect(element).toHaveClass('active');
expect(element).toHaveAttribute('href', '/home');
expect(element).toBeChecked();
expect(element).toHaveFocus();
```

### Vitest

Vitest is a Vite-native test runner. It is compatible with Jest's API but runs faster because it reuses Vite's transformation pipeline.

```bash
npm install --save-dev vitest @vitest/ui jsdom
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.ts',
  },
});
```

```ts
// src/test/setup.ts
import '@testing-library/jest-dom';
```

Key differences from Jest:

| Feature | Jest | Vitest |
|---|---|---|
| Speed | Moderate | Faster (Vite HMR pipeline) |
| Config | Separate jest.config | Inside vite.config |
| ESM support | Requires Babel transform | Native ESM |
| Watch mode | jest --watch | vitest (default interactive) |
| API compatibility | Jest API | Jest-compatible |
| TypeScript | Requires ts-jest | Native |

### Playwright

End-to-end testing against real browsers (Chromium, Firefox, WebKit).

```bash
npm install --save-dev @playwright/test
npx playwright install
```

```ts
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test('user can log in', async ({ page }) => {
  await page.goto('http://localhost:3000/login');
  await page.getByLabel('Email').fill('user@example.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: /log in/i }).click();
  await expect(page).toHaveURL('/dashboard');
});
```

### Cypress

Alternative E2E framework with an interactive GUI runner.

```bash
npm install --save-dev cypress
npx cypress open
```

| Feature | Playwright | Cypress |
|---|---|---|
| Browsers | Chrome, Firefox, WebKit | Chrome, Firefox (limited) |
| Speed | Faster parallel execution | Moderate |
| Parallel testing | Built-in | Requires Cypress Cloud |
| API style | Async/await (standard) | Custom thenable chain |
| Component testing | Yes | Yes |
| Network interception | Yes (route) | Yes (intercept) |

---

## 3. React Testing Library Philosophy

### Test Behavior, Not Implementation

RTL was designed around a single guiding principle:

> "The more your tests resemble the way your software is used, the more confidence they can give you." — Kent C. Dodds

This means:
- Do NOT access component instance state directly
- Do NOT call internal component methods
- Do NOT assert that a specific internal function was called (unless it is a critical side effect exposed as a prop)
- DO render the component, interact with it as a user would, and assert on visible DOM output

### ❌ Wrong — Testing Implementation Details

```jsx
import { shallow } from 'enzyme';
import Counter from './Counter';

test('increments internal state', () => {
  const wrapper = shallow(<Counter />);
  wrapper.instance().handleIncrement(); // calling internal method
  expect(wrapper.state('count')).toBe(1); // reading internal state
});
```

This test will break if you rename `handleIncrement` or move count to a different state shape, even if the component still works correctly from a user's perspective.

### ✅ Correct — Testing Observable Behavior

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Counter from './Counter';

test('shows updated count after increment click', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  await user.click(screen.getByRole('button', { name: /increment/i }));

  expect(screen.getByText('1')).toBeInTheDocument();
});
```

This test survives internal refactoring because it only cares about what the user sees.

### Query by What Users See

Users interact with text, roles, labels, and placeholders — not with CSS class names or component names.

Priority order enforced by RTL's guiding philosophy (highest to lowest):

1. `getByRole` — ARIA roles (button, heading, link, textbox, checkbox)
2. `getByLabelText` — form element labels
3. `getByPlaceholderText` — input placeholder text
4. `getByText` — visible text content
5. `getByDisplayValue` — current value of form element
6. `getByAltText` — image alt attribute
7. `getByTitle` — title attribute
8. `getByTestId` — last resort: `data-testid` attribute

### Avoid Reaching into Internal State

If you find yourself writing `wrapper.state()`, `component.instance().myProp`, or similar, you are testing implementation details. Rethink the test to drive behavior through the UI instead.

---

## 4. Basic Test Structure

### describe, it / test, expect

```jsx
// Greeting.test.jsx
import { render, screen } from '@testing-library/react';
import Greeting from './Greeting';

describe('Greeting component', () => {
  it('renders a greeting with the provided name', () => {
    render(<Greeting name="Alice" />);
    expect(screen.getByText('Hello, Alice!')).toBeInTheDocument();
  });

  it('renders a fallback when no name is provided', () => {
    render(<Greeting />);
    expect(screen.getByText('Hello, Stranger!')).toBeInTheDocument();
  });

  describe('when name is empty string', () => {
    it('renders the fallback greeting', () => {
      render(<Greeting name="" />);
      expect(screen.getByText('Hello, Stranger!')).toBeInTheDocument();
    });
  });
});
```

- `describe` groups related tests. Nesting is permitted and useful for organizing by scenario.
- `it` and `test` are interchangeable. `it` reads more naturally ("it renders...").
- `expect(value).matcher()` is the assertion syntax.

### render from @testing-library/react

`render` mounts the component into a jsdom DOM. Its return value includes utilities, but the preferred way to query is via the global `screen` object.

```jsx
const { container, unmount, rerender, baseElement } = render(
  <MyComponent prop="value" />
);

// Rerender with different props (same component instance)
rerender(<MyComponent prop="new-value" />);

// Manually unmount (RTL's afterEach cleanup handles this automatically)
unmount();
```

### screen queries

`screen` is a global object that queries against the currently rendered DOM. Always prefer `screen` over destructuring queries from the `render` return value.

```jsx
// Preferred: screen queries
screen.getByRole('button', { name: /submit/i });
screen.getByLabelText('Email address');
screen.getByPlaceholderText('Search...');
screen.getByText(/welcome/i);
screen.queryByText('Error message'); // null if not found
await screen.findByText('Loaded!');  // async: waits up to 1s
```

### Automatic Cleanup

RTL registers `cleanup` with `afterEach` automatically when you import from `@testing-library/react`. This unmounts the component and clears the jsdom DOM between tests.

```js
// This is automatic — you do NOT need to call cleanup() yourself
// unless you are using the /pure import
import '@testing-library/react'; // auto-registers cleanup
```

### userEvent and fireEvent

```jsx
import userEvent from '@testing-library/user-event';
import { fireEvent, render, screen } from '@testing-library/react';

// userEvent (preferred — realistic event sequences)
const user = userEvent.setup();
await user.click(screen.getByRole('button'));
await user.type(screen.getByRole('textbox'), 'hello');
await user.clear(screen.getByRole('textbox'));

// fireEvent (lower-level — fires a single synthetic event)
fireEvent.click(screen.getByRole('button'));
fireEvent.change(screen.getByRole('textbox'), { target: { value: 'hello' } });
```

---

## 5. Queries

### getBy

Throws an error immediately if zero or more than one element matches. Use when you are certain the element must exist and must be unique.

```jsx
// Throws: "Unable to find an accessible element with the role 'button'..."
const button = screen.getByRole('button', { name: /submit/i });

// Throws: "Found multiple elements with the role 'heading'"
const heading = screen.getByRole('heading'); // fails if multiple <h1>-<h6>
```

### queryBy

Returns `null` if the element is not found. Throws if multiple elements match. Use to assert absence.

```jsx
// Returns null if element is absent (no throw)
const errorMessage = screen.queryByText(/error/i);
expect(errorMessage).toBeNull();
expect(errorMessage).not.toBeInTheDocument();

// ❌ Wrong — using getBy to assert absence
expect(() => screen.getByText(/error/i)).toThrow(); // anti-pattern

// ✅ Correct
expect(screen.queryByText(/error/i)).not.toBeInTheDocument();
```

### findBy

Returns a Promise that resolves when the element appears. Retries until the element appears or the timeout (default 1000ms) expires. Use for async rendering.

```jsx
// Waits up to 1000ms for the element to appear
const successMessage = await screen.findByText('Data loaded successfully');
expect(successMessage).toBeInTheDocument();
```

Custom timeout:

```jsx
await screen.findByText('Loaded', {}, { timeout: 3000 });
```

### getAllBy / queryAllBy / findAllBy

Return arrays of all matching elements.

```jsx
// Throws if no match
const items = screen.getAllByRole('listitem');
expect(items).toHaveLength(5);

// Returns empty array if no match (no throw)
const inputs = screen.queryAllByRole('textbox');
expect(inputs).toHaveLength(0);

// Async — waits for at least one match
const cards = await screen.findAllByRole('article');
```

### Full Query Priority Reference

| Priority | Query | Use Case |
|---|---|---|
| 1 | `getByRole` | Buttons, headings, links, inputs, checkboxes |
| 2 | `getByLabelText` | Form fields associated with `<label>` |
| 3 | `getByPlaceholderText` | Inputs with placeholder attribute |
| 4 | `getByText` | Paragraphs, spans, non-interactive text |
| 5 | `getByDisplayValue` | Input/select showing a current value |
| 6 | `getByAltText` | Images with alt attribute |
| 7 | `getByTitle` | Elements with title attribute |
| 8 | `getByTestId` | Last resort — use `data-testid` sparingly |

### ARIA Role Reference

```txt
button      → <button>, <input type="button|submit|reset">
link        → <a href="...">
heading     → <h1> through <h6>
textbox     → <input type="text">, <textarea>
checkbox    → <input type="checkbox">
radio       → <input type="radio">
combobox    → <select>, elements with role="combobox"
listitem    → <li>
list        → <ul>, <ol>
img         → <img>
dialog      → <dialog>, role="dialog"
alert       → role="alert"
navigation  → <nav>
main        → <main>
banner      → <header>
contentinfo → <footer>
```

### Accessible Name Matching

`getByRole` accepts a `name` option to distinguish elements with the same role. The accessible name comes from label, aria-label, aria-labelledby, or button content.

```jsx
// Two buttons in the DOM
screen.getByRole('button', { name: /save/i });
screen.getByRole('button', { name: /cancel/i });

// Case-insensitive matching with regex
screen.getByRole('heading', { name: /react testing/i });
```

---

## 6. User Interactions

### userEvent.setup()

In `@testing-library/user-event` v14, initialize a user instance at the start of each test. The instance maintains pointer and keyboard state across multiple interactions.

```jsx
import userEvent from '@testing-library/user-event';

describe('Form tests', () => {
  it('fills and submits the form', async () => {
    const user = userEvent.setup();

    render(<ContactForm onSubmit={jest.fn()} />);

    await user.type(screen.getByLabelText(/name/i), 'Alice');
    await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
    await user.click(screen.getByRole('button', { name: /send/i }));
  });
});
```

Configuration options for `setup()`:

```jsx
const user = userEvent.setup({
  delay: null,        // no delay between keystrokes (faster tests)
  pointerEventsCheck: 0, // skip pointer-events:none check
});
```

### userEvent.click()

Fires a complete click sequence: pointerover, pointerenter, mouseover, mouseenter, pointermove, mousemove, pointerdown, mousedown, focus, pointerup, mouseup, click.

```jsx
test('calls onClick handler', async () => {
  const handleClick = jest.fn();
  const user = userEvent.setup();

  render(<button onClick={handleClick}>Click me</button>);
  await user.click(screen.getByRole('button', { name: /click me/i }));

  expect(handleClick).toHaveBeenCalledTimes(1);
});
```

### userEvent.type()

Simulates typing character by character, firing `keydown`, `keypress`, `input`, `keyup` for each character. Handles special keys with curly-brace syntax.

```jsx
test('updates input as user types', async () => {
  const user = userEvent.setup();
  render(<input type="text" aria-label="Search" />);

  const input = screen.getByRole('textbox', { name: /search/i });
  await user.type(input, 'hello world');

  expect(input).toHaveValue('hello world');
});
```

Special key syntax:

```jsx
await user.type(input, '{Enter}');
await user.type(input, '{Backspace}');
await user.type(input, '{Tab}');
await user.type(input, 'hello{selectall}{backspace}'); // type, select all, delete
```

### userEvent.clear()

Selects all text in an input and deletes it.

```jsx
await user.clear(screen.getByRole('textbox'));
expect(screen.getByRole('textbox')).toHaveValue('');
```

### userEvent.selectOptions()

For `<select>` elements:

```jsx
await user.selectOptions(
  screen.getByRole('combobox'),
  screen.getByRole('option', { name: 'Banana' })
);
expect(screen.getByRole('combobox')).toHaveValue('banana');
```

### userEvent.keyboard()

Fires keyboard events without a target element. Useful for keyboard shortcuts.

```jsx
await user.keyboard('{Enter}');
await user.keyboard('{Tab}');
await user.keyboard('{Escape}');
await user.keyboard('{Control>}a{/Control}'); // Ctrl+A
await user.keyboard('{Shift>}{Tab}{/Shift}'); // Shift+Tab
```

### fireEvent vs userEvent

| Aspect | fireEvent | userEvent |
|---|---|---|
| Realism | Single synthetic event | Full browser event sequence |
| Asynchronous | Synchronous | Returns Promise (must await) |
| Focus management | Does not handle focus | Manages focus correctly |
| Pointer events | Skipped | Included |
| Default choice | No | Yes |
| When to use | Edge cases needing direct control | All normal interaction tests |

---

## 7. Async Testing

### waitFor

`waitFor` repeatedly calls its callback function until it stops throwing or a timeout expires. Use it to wait for an async DOM change that is triggered by a non-query action.

```jsx
import { render, screen, waitFor } from '@testing-library/react';

test('shows error message after failed API call', async () => {
  server.use(
    rest.get('/api/data', (req, res, ctx) => res(ctx.status(500)))
  );

  render(<DataFetcher />);

  await waitFor(() => {
    expect(screen.getByRole('alert')).toHaveTextContent('Failed to load');
  });
});
```

`waitFor` options:

```jsx
await waitFor(
  () => expect(screen.getByText('Done')).toBeInTheDocument(),
  {
    timeout: 3000,   // max wait in ms (default: 1000)
    interval: 100,   // retry interval in ms (default: 50)
  }
);
```

### findBy (built-in async query)

`findBy` is syntactic sugar for `waitFor(() => getBy(...))`. Always prefer `findBy` over manually combining `waitFor` + `getBy` for simple async queries.

```jsx
// Equivalent — prefer findBy:
const el = await screen.findByText('Loaded');

// Manual equivalent (more verbose):
await waitFor(() => screen.getByText('Loaded'));
const el = screen.getByText('Loaded');
```

### Testing Data Fetching

```jsx
// Component
function UserCard({ userId }) {
  const [user, setUser] = React.useState(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState(null);

  React.useEffect(() => {
    const controller = new AbortController();
    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(res => {
        if (!res.ok) throw new Error('Failed');
        return res.json();
      })
      .then(data => { setUser(data); setLoading(false); })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err.message);
          setLoading(false);
        }
      });
    return () => controller.abort();
  }, [userId]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p role="alert">{error}</p>;
  return <h1>{user.name}</h1>;
}

// Test
test('shows loading then user name', async () => {
  render(<UserCard userId="1" />);

  expect(screen.getByText('Loading...')).toBeInTheDocument();

  const name = await screen.findByRole('heading');
  expect(name).toHaveTextContent('Alice');
});

test('shows error on failed fetch', async () => {
  server.use(
    rest.get('/api/users/:id', (req, res, ctx) => res(ctx.status(500)))
  );

  render(<UserCard userId="1" />);

  const alert = await screen.findByRole('alert');
  expect(alert).toHaveTextContent('Failed');
});
```

### act() and Async State Updates

React wraps state updates in `act()` to flush effects and state updates synchronously. RTL automatically wraps all of its utilities in `act()`, but direct hook calls inside `renderHook` require manual wrapping.

```jsx
import { act } from '@testing-library/react';

// With fake timers
jest.useFakeTimers();

render(<AutoSave interval={3000} />);

act(() => {
  jest.advanceTimersByTime(3000);
});

expect(screen.getByText('Saved')).toBeInTheDocument();

jest.useRealTimers();
```

### Testing with Fake Timers

```jsx
test('debounced search fires after delay', async () => {
  jest.useFakeTimers();
  const user = userEvent.setup({ delay: null }); // disable userEvent's own delay
  const handleSearch = jest.fn();

  render(<SearchInput onSearch={handleSearch} />);

  await user.type(screen.getByRole('textbox'), 'react');

  expect(handleSearch).not.toHaveBeenCalled(); // debounce hasn't fired

  act(() => { jest.advanceTimersByTime(500); });

  expect(handleSearch).toHaveBeenCalledWith('react');

  jest.useRealTimers();
});
```

---

## 8. Mocking

### jest.fn()

Creates a mock function that records all calls, arguments, and return values.

```js
const mockFn = jest.fn();

mockFn('hello');
mockFn('world');

expect(mockFn).toHaveBeenCalledTimes(2);
expect(mockFn).toHaveBeenCalledWith('hello');
expect(mockFn).toHaveBeenNthCalledWith(1, 'hello');
expect(mockFn).toHaveBeenLastCalledWith('world');
```

Setting return values:

```js
const mockFn = jest.fn();

mockFn.mockReturnValue(42);                            // always returns 42
mockFn.mockResolvedValue({ data: 'result' });          // resolves Promise
mockFn.mockRejectedValue(new Error('Network error'));  // rejects Promise

// Per-call overrides:
mockFn
  .mockReturnValueOnce('first call')
  .mockReturnValueOnce('second call')
  .mockReturnValue('all subsequent calls');
```

Custom implementation:

```js
const mockFn = jest.fn((x) => x * 2);
expect(mockFn(5)).toBe(10);
```

### jest.mock()

Replaces an entire module with an auto-mock or manual implementation. Jest hoists `jest.mock()` calls to the top of the file automatically.

```js
// Auto-mock: all exports become jest.fn()
jest.mock('./utils');

// Manual mock with specific implementation:
jest.mock('./api', () => ({
  fetchUser: jest.fn().mockResolvedValue({ id: 1, name: 'Alice' }),
  updateUser: jest.fn().mockResolvedValue({ success: true }),
}));
```

❌ Wrong — mock inside test body:
```js
test('bad pattern', () => {
  jest.mock('./api'); // too late — not hoisted, does not work
});
```

✅ Correct — mock at module scope:
```js
jest.mock('./api', () => ({
  fetchUser: jest.fn(),
}));

import { fetchUser } from './api';

beforeEach(() => {
  fetchUser.mockResolvedValue({ name: 'Bob' });
});
```

### jest.spyOn()

Creates a mock that wraps an existing implementation, allowing you to assert calls while preserving (or overriding) the original behavior.

```js
const consoleSpy = jest.spyOn(console, 'error').mockImplementation(() => {});

// Run code that calls console.error

expect(consoleSpy).toHaveBeenCalledWith(expect.stringContaining('Error'));
consoleSpy.mockRestore(); // restore original console.error
```

### Mocking fetch Directly

```js
global.fetch = jest.fn();

beforeEach(() => {
  fetch.mockResolvedValue({
    ok: true,
    json: jest.fn().mockResolvedValue({ users: [] }),
  });
});

afterEach(() => {
  jest.clearAllMocks();
});
```

### MSW (Mock Service Worker)

MSW intercepts HTTP requests at the Service Worker / Node.js level. It is framework-agnostic, works with any HTTP client (fetch, axios, react-query), and can be used in both Jest (via `msw/node`) and the browser.

```bash
npm install --save-dev msw
```

```js
// src/mocks/handlers.js
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json([
        { id: 1, name: 'Alice' },
        { id: 2, name: 'Bob' },
      ])
    );
  }),

  rest.post('/api/login', async (req, res, ctx) => {
    const { email } = await req.json();
    if (email === 'wrong@example.com') {
      return res(ctx.status(401), ctx.json({ error: 'Invalid credentials' }));
    }
    return res(ctx.json({ token: 'fake-jwt', user: { email } }));
  }),
];
```

```js
// src/mocks/server.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```js
// jest.setup.js
import { server } from './src/mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());  // reset per-test overrides
afterAll(() => server.close());
```

Overriding handlers for a specific test:

```js
test('handles empty user list', async () => {
  server.use(
    rest.get('/api/users', (req, res, ctx) => {
      return res(ctx.json([]));
    })
  );

  render(<UserList />);
  expect(await screen.findByText('No users found')).toBeInTheDocument();
});
```

---

## 9. Testing Custom Hooks

### renderHook

`renderHook` mounts a minimal component whose sole purpose is calling your hook and exposing its return value.

```jsx
import { renderHook, act } from '@testing-library/react';
import useCounter from './useCounter';

// useCounter implementation:
// function useCounter(initial = 0) {
//   const [count, setCount] = useState(initial);
//   const increment = () => setCount(c => c + 1);
//   const decrement = () => setCount(c => c - 1);
//   const reset = () => setCount(initial);
//   return { count, increment, decrement, reset };
// }

test('initializes with default value 0', () => {
  const { result } = renderHook(() => useCounter());
  expect(result.current.count).toBe(0);
});

test('initializes with custom value', () => {
  const { result } = renderHook(() => useCounter(10));
  expect(result.current.count).toBe(10);
});

test('increments count', () => {
  const { result } = renderHook(() => useCounter());

  act(() => {
    result.current.increment();
  });

  expect(result.current.count).toBe(1);
});

test('resets to initial value', () => {
  const { result } = renderHook(() => useCounter(5));

  act(() => {
    result.current.increment();
    result.current.increment();
    result.current.reset();
  });

  expect(result.current.count).toBe(5);
});
```

### result.current

`result` is a ref-like object. `result.current` always points to the most recent return value of the hook. After `act()`, it is updated synchronously.

### Testing a Hook with Changing Props

```jsx
const { result, rerender } = renderHook(
  ({ step }) => useCounter(0, step),
  { initialProps: { step: 1 } }
);

act(() => { result.current.increment(); });
expect(result.current.count).toBe(1);

// Rerender with new props
rerender({ step: 5 });

act(() => { result.current.increment(); });
expect(result.current.count).toBe(6); // 1 + 5
```

### act() Requirement for State Updates

Any function call that triggers a React state update outside of RTL's built-in utilities must be wrapped in `act()`.

```jsx
// ❌ Wrong — React warning: state update not wrapped in act()
result.current.increment();
expect(result.current.count).toBe(1);

// ✅ Correct
act(() => {
  result.current.increment();
});
expect(result.current.count).toBe(1);
```

### Hooks That Use Context

```jsx
import { renderHook } from '@testing-library/react';
import { AuthContext } from './AuthContext';
import useCurrentUser from './useCurrentUser';

const fakeUser = { id: 1, name: 'Alice', role: 'admin' };

const wrapper = ({ children }) => (
  <AuthContext.Provider value={{ user: fakeUser, logout: jest.fn() }}>
    {children}
  </AuthContext.Provider>
);

test('returns current user from context', () => {
  const { result } = renderHook(() => useCurrentUser(), { wrapper });
  expect(result.current).toEqual(fakeUser);
});
```

### Testing Async Hooks

```jsx
import { renderHook, waitFor } from '@testing-library/react';
import useUserData from './useUserData';

test('transitions from loading to loaded', async () => {
  const { result } = renderHook(() => useUserData('user-1'));

  // Initial state
  expect(result.current.loading).toBe(true);
  expect(result.current.data).toBeNull();

  // Wait for fetch to complete
  await waitFor(() => {
    expect(result.current.loading).toBe(false);
  });

  expect(result.current.data).toEqual({ id: 'user-1', name: 'Alice' });
  expect(result.current.error).toBeNull();
});
```

---

## 10. Component Testing Patterns

### Testing a Button Click

```jsx
// ToggleButton.jsx
function ToggleButton() {
  const [on, setOn] = React.useState(false);
  return (
    <button onClick={() => setOn(prev => !prev)}>
      {on ? 'ON' : 'OFF'}
    </button>
  );
}

// ToggleButton.test.jsx
test('toggles between ON and OFF on click', async () => {
  const user = userEvent.setup();
  render(<ToggleButton />);

  const button = screen.getByRole('button');
  expect(button).toHaveTextContent('OFF');

  await user.click(button);
  expect(button).toHaveTextContent('ON');

  await user.click(button);
  expect(button).toHaveTextContent('OFF');
});
```

### Testing Form Submission

```jsx
// LoginForm.jsx
function LoginForm({ onSubmit }) {
  const [email, setEmail] = React.useState('');
  const [password, setPassword] = React.useState('');
  const [error, setError] = React.useState('');

  function handleSubmit(e) {
    e.preventDefault();
    if (!email) { setError('Email is required'); return; }
    onSubmit({ email, password });
  }

  return (
    <form onSubmit={handleSubmit}>
      {error && <p role="alert">{error}</p>}
      <label>
        Email
        <input type="email" value={email} onChange={e => setEmail(e.target.value)} />
      </label>
      <label>
        Password
        <input type="password" value={password} onChange={e => setPassword(e.target.value)} />
      </label>
      <button type="submit">Log In</button>
    </form>
  );
}

// LoginForm.test.jsx
test('submits email and password to handler', async () => {
  const handleSubmit = jest.fn();
  const user = userEvent.setup();

  render(<LoginForm onSubmit={handleSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'alice@example.com');
  await user.type(screen.getByLabelText(/password/i), 'secret123');
  await user.click(screen.getByRole('button', { name: /log in/i }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'alice@example.com',
    password: 'secret123',
  });
});

test('shows validation error when email is empty', async () => {
  const handleSubmit = jest.fn();
  const user = userEvent.setup();

  render(<LoginForm onSubmit={handleSubmit} />);
  await user.click(screen.getByRole('button', { name: /log in/i }));

  expect(screen.getByRole('alert')).toHaveTextContent('Email is required');
  expect(handleSubmit).not.toHaveBeenCalled();
});
```

### Testing Conditional Rendering

```jsx
// Alert.jsx
function Alert({ type, message }) {
  if (!message) return null;
  return <div role="alert" data-type={type}>{message}</div>;
}

// Alert.test.jsx
test('renders nothing when message is absent', () => {
  render(<Alert type="error" />);
  expect(screen.queryByRole('alert')).not.toBeInTheDocument();
});

test('renders error message when provided', () => {
  render(<Alert type="error" message="Something went wrong" />);
  expect(screen.getByRole('alert')).toHaveTextContent('Something went wrong');
});
```

### Testing List Rendering

```jsx
// UserList.jsx
function UserList({ users }) {
  if (users.length === 0) return <p>No users found.</p>;
  return (
    <ul aria-label="User list">
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}

// UserList.test.jsx
test('renders all users in the list', () => {
  const users = [
    { id: 1, name: 'Alice' },
    { id: 2, name: 'Bob' },
    { id: 3, name: 'Charlie' },
  ];

  render(<UserList users={users} />);

  const items = screen.getAllByRole('listitem');
  expect(items).toHaveLength(3);
  expect(items[0]).toHaveTextContent('Alice');
  expect(items[1]).toHaveTextContent('Bob');
  expect(items[2]).toHaveTextContent('Charlie');
});

test('shows empty state when no users', () => {
  render(<UserList users={[]} />);
  expect(screen.getByText('No users found.')).toBeInTheDocument();
  expect(screen.queryAllByRole('listitem')).toHaveLength(0);
});
```

---

## 11. Testing with Context

### Wrapping Component in Provider

When a component consumes context, the test must supply that context via a Provider wrapper.

```jsx
// ThemeButton.jsx
import { useContext } from 'react';
import { ThemeContext } from './ThemeContext';

function ThemeButton() {
  const { theme, toggleTheme } = useContext(ThemeContext);
  return (
    <button onClick={toggleTheme} aria-pressed={theme === 'dark'}>
      Current: {theme}
    </button>
  );
}

// ThemeButton.test.jsx
import { ThemeContext } from './ThemeContext';

test('displays current theme from context', () => {
  const toggleTheme = jest.fn();
  render(
    <ThemeContext.Provider value={{ theme: 'dark', toggleTheme }}>
      <ThemeButton />
    </ThemeContext.Provider>
  );

  expect(screen.getByRole('button')).toHaveTextContent('Current: dark');
});

test('calls toggleTheme when clicked', async () => {
  const toggleTheme = jest.fn();
  const user = userEvent.setup();

  render(
    <ThemeContext.Provider value={{ theme: 'light', toggleTheme }}>
      <ThemeButton />
    </ThemeContext.Provider>
  );

  await user.click(screen.getByRole('button'));
  expect(toggleTheme).toHaveBeenCalledTimes(1);
});
```

### Custom render Helper with Providers

When most tests require the same set of providers, create a custom `render` function.

```jsx
// src/test-utils.jsx
import { render } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { ThemeProvider } from './context/ThemeContext';
import { AuthProvider } from './context/AuthContext';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,          // don't retry failed queries in tests
        gcTime: Infinity,      // don't garbage collect during tests
      },
    },
  });
}

function AllProviders({ children }) {
  const queryClient = createTestQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      <MemoryRouter>
        <AuthProvider>
          <ThemeProvider>
            {children}
          </ThemeProvider>
        </AuthProvider>
      </MemoryRouter>
    </QueryClientProvider>
  );
}

const customRender = (ui, options = {}) =>
  render(ui, { wrapper: AllProviders, ...options });

// Re-export everything from RTL so tests only need one import
export * from '@testing-library/react';
export { customRender as render };
```

Usage:

```jsx
// In any test file — providers are applied automatically
import { render, screen } from '../test-utils';

test('dashboard renders user name', async () => {
  render(<Dashboard />);
  expect(await screen.findByText('Welcome, Alice')).toBeInTheDocument();
});
```

---

## 12. Snapshot Testing

### What Snapshot Testing Does

A snapshot test serializes the rendered output of a component to a `.snap` file. On subsequent runs it diffs the current output against the saved snapshot. If they differ, the test fails.

```jsx
import { render } from '@testing-library/react';
import Badge from './Badge';

test('renders primary badge correctly', () => {
  const { container } = render(<Badge variant="primary">New</Badge>);
  expect(container).toMatchSnapshot();
});
```

First run creates `__snapshots__/Badge.test.jsx.snap`:

```txt
// Jest Snapshot v1, https://goo.gl/fbAQLP

exports[`renders primary badge correctly 1`] = `
<div>
  <span
    class="badge badge-primary"
  >
    New
  </span>
</div>
`;
```

### When to Use Snapshots

✅ Correct use cases:
- Capturing stable output of purely presentational, atomic components (Badge, Button, Avatar)
- Detecting unintentional markup changes
- Components with very few or no dynamic parts

❌ Wrong use cases:
- Components with timestamps, random IDs, or other dynamic data
- Large page-level components (snapshot becomes a 500-line file nobody reads)
- As a substitute for behavioral tests
- Snapshot of the entire application tree

### Updating Snapshots

```bash
# Update all snapshots that changed intentionally
npx jest --updateSnapshot

# Update snapshots for a single test file
npx jest Badge.test.jsx --updateSnapshot

# Shorthand flag
npx jest -u
```

Always review diffs before updating. A changed snapshot diff in a PR is a signal to reviewers that markup changed intentionally.

### Inline Snapshots

Store the snapshot string directly in the test file rather than a separate `.snap` file.

```jsx
test('renders badge', () => {
  const { container } = render(<Badge variant="info">Beta</Badge>);
  expect(container).toMatchInlineSnapshot(`
    <div>
      <span
        class="badge badge-info"
      >
        Beta
      </span>
    </div>
  `);
});
```

Jest writes the snapshot string into the test file on first run.

---

## 13. Debugging Techniques

### React DevTools: Components Tab

The Components tab in React DevTools shows the full component tree and lets you inspect:
- Props and state for every component
- Context values
- Hook values (useState, useReducer, useRef, custom hooks)
- Owner information (which component rendered this one)

Useful actions:
- Click any component node to inspect it in the panel
- Edit props and state live to test different states without code changes
- Toggle "Highlight updates when components render" to see re-renders visually
- Navigate to the source file from the component panel (requires source maps)

### React DevTools: Profiler Tab

```text
Click "Record"
        ↓
Interact with the app (click button, type, navigate)
        ↓
Click "Stop"
        ↓
Select a commit in the timeline
        ↓
Analyze flame chart (width = render time)
        ↓
Click a component bar → "Why did this render?"
        ↓
Fix: React.memo, useMemo, useCallback
```

### console.log Placement Strategy

```jsx
function ProductCard({ product }) {
  console.log('[ProductCard] render', product.id); // logs every render

  useEffect(() => {
    console.log('[ProductCard] mounted or product changed', product.id);
  }, [product]);

  const price = useMemo(() => {
    console.log('[ProductCard] recomputing price');
    return formatPrice(product.price);
  }, [product.price]);

  return <div>{price}</div>;
}
```

Rules:
- Prefix with `[ComponentName]` for easy filtering in the console
- Remove all `console.log` before committing (enforce via ESLint `no-console`)
- Use `console.warn` for intentional deprecation warnings
- Use `console.error` for legitimate error conditions

### Browser Breakpoints

To debug a React component in Chrome DevTools:
1. Open Sources panel
2. Press `Ctrl+P` (or `Cmd+P`) and type the component file name
3. Click the line number to add a breakpoint
4. Interact with the app to hit the breakpoint
5. In the right panel: inspect Scope, Call Stack, Watch expressions
6. Use Step Over (F10), Step Into (F11), Step Out (Shift+F11)

Conditional breakpoints:
1. Right-click a line number → "Add conditional breakpoint"
2. Enter a condition: `userId === 'admin'`

Logpoints (print without stopping):
1. Right-click a line number → "Add logpoint"
2. Enter: `'User rendered: ' + user.name`

### React DevTools "Why did this render?"

Enable in DevTools Settings → Profiler → "Record why each component rendered while profiling." During a recorded profile, selecting a component shows one of:

```txt
- Props changed: prop "items" changed
- State changed: hook index 0 changed
- Context changed: ThemeContext changed
- Parent component re-rendered
- This is the first time this component rendered
```

### Strict Mode Double Rendering

In development, `<React.StrictMode>` intentionally renders components twice to detect side effects. This is not a bug — it is intentional:

```jsx
// Any log you see twice in development is from StrictMode double-render
function App() {
  return (
    <React.StrictMode>
      <Router />
    </React.StrictMode>
  );
}
```

---

## 14. Common React Bugs

### Infinite Re-render

Symptom: "Too many re-renders. React limits the number of renders to prevent an infinite loop."

```jsx
// ❌ Wrong — state setter called unconditionally during render
function BadComponent() {
  const [count, setCount] = React.useState(0);
  setCount(count + 1); // called on every render → triggers another render → infinite loop
  return <div>{count}</div>;
}
```

```jsx
// ✅ Correct — state updates inside event handlers or effects
function GoodComponent() {
  const [count, setCount] = React.useState(0);
  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

### Memory Leak (Async in Unmounted Component)

Symptom: "Warning: Can't perform a React state update on an unmounted component."

```jsx
// ❌ Wrong — fetch callback may run after component unmounts
function BadFetcher({ userId }) {
  const [user, setUser] = React.useState(null);

  React.useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data)); // may run after unmount
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

```jsx
// ✅ Correct — AbortController cancels the fetch on cleanup
function GoodFetcher({ userId }) {
  const [user, setUser] = React.useState(null);

  React.useEffect(() => {
    const controller = new AbortController();

    fetch(`/api/users/${userId}`, { signal: controller.signal })
      .then(res => res.json())
      .then(data => setUser(data))
      .catch(err => {
        if (err.name !== 'AbortError') console.error(err);
      });

    return () => controller.abort();
  }, [userId]);

  return <div>{user?.name}</div>;
}
```

### Stale Closure

Symptom: A function inside `useEffect` or `useCallback` reads an outdated value of state.

```jsx
// ❌ Wrong — count is always 0 inside the interval
function BadCounter() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    const id = setInterval(() => {
      setCount(count + 1); // 'count' is captured as 0 at setup time
    }, 1000);
    return () => clearInterval(id);
  }, []); // empty deps — effect never re-runs, count is stale

  return <div>{count}</div>; // always shows 1 (0+1) after first tick, never increments further
}
```

```jsx
// ✅ Correct — functional updater accesses latest state
function GoodCounter() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    const id = setInterval(() => {
      setCount(c => c + 1); // always reads latest value via updater function
    }, 1000);
    return () => clearInterval(id);
  }, []);

  return <div>{count}</div>;
}
```

### Missing key Warning

```txt
Warning: Each child in a list should have a unique "key" prop.
```

```jsx
// ❌ Wrong — no key, or using unstable index as key with dynamic lists
{items.map((item, index) => (
  <TodoItem key={index} item={item} />  // problematic if items are reordered/deleted
))}
```

```jsx
// ✅ Correct — stable unique key from data
{items.map(item => (
  <TodoItem key={item.id} item={item} />
))}
```

### React key Causing Accidental State Reset

When a component's `key` prop changes, React destroys the old instance and mounts a fresh one. This resets all internal state, refs, and effects.

```jsx
// Intentional reset: new userId → new form (no carry-over of old user's edits)
<UserEditForm key={selectedUserId} userId={selectedUserId} />

// Unintentional reset: key changes on every render
<SearchInput key={Date.now()} /> // ❌ resets on every parent re-render
```

---

## 15. Error Boundaries for Debugging

### What Error Boundaries Are

Error boundaries are class components that implement `getDerivedStateFromError` and/or `componentDidCatch`. They intercept render-phase JavaScript errors in their child tree and display a fallback UI instead of crashing the entire application.

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error) {
    // Update state so the next render shows the fallback
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log to an error tracking service like Sentry
    logErrorToService(error, errorInfo.componentStack);
    this.setState({ errorInfo });
  }

  render() {
    if (this.state.hasError) {
      return (
        <div role="alert" className="error-boundary">
          <h2>Something went wrong.</h2>
          {process.env.NODE_ENV === 'development' && (
            <details>
              <summary>Error details (development only)</summary>
              <pre>{this.state.error?.toString()}</pre>
              <pre>{this.state.errorInfo?.componentStack}</pre>
            </details>
          )}
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

### Usage in Component Tree

```jsx
function App() {
  return (
    <ErrorBoundary>
      <Header />
      <main>
        <ErrorBoundary fallback={<WidgetError />}>
          <DataWidget />
        </ErrorBoundary>
        <ErrorBoundary fallback={<ChartError />}>
          <AnalyticsChart />
        </ErrorBoundary>
      </main>
    </ErrorBoundary>
  );
}
```

### What Error Boundaries Do NOT Catch

| Error Type | Caught by Error Boundary? |
|---|---|
| Render method error | ✅ Yes |
| Constructor error | ✅ Yes |
| getDerivedStateFromProps error | ✅ Yes |
| Event handler error | ❌ No — use try/catch |
| Async code (setTimeout, fetch) | ❌ No — use try/catch |
| Server-side rendering | ❌ No |
| Error in the error boundary itself | ❌ No |

---

## 16. Using React DevTools Profiler

### Flame Chart

The flame chart represents each component as a horizontal bar. Bar width corresponds to render duration. Bars are stacked by component hierarchy.

```text
App                    ███████████████████████████████████ 25ms
  Router               ██████████████████████████████ 22ms
    Dashboard          ███████████████████████████ 20ms
      Sidebar          ████ 3ms
      MainContent      █████████████████████ 16ms
        DataTable      ████████████████ 12ms
          Row (x50)    ██ 0.2ms each
```

Click any bar to see:
- Component name
- Render time (this commit)
- Why it rendered

### Identifying Unnecessary Re-renders

Steps:
1. Enable "Record why each component rendered" in Profiler settings
2. Record a user interaction
3. Look for components that rendered but show "Parent component re-rendered" as the only reason
4. These are candidates for `React.memo` wrapping

```jsx
// Component re-renders every time parent re-renders even if its props didn't change
const ExpensiveList = React.memo(function ExpensiveList({ items }) {
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
});
```

### Commit Timeline

Each vertical bar in the top of the Profiler represents one React commit (one batch of DOM updates). Color intensity indicates duration:
- Gray: fast commit
- Yellow/orange: moderate
- Red: slow commit

Click a commit to see which components rendered in it and why.

### Measuring Component Render Count

```jsx
// Add this in development to count renders
let renderCount = 0;

function ProfiledComponent() {
  if (process.env.NODE_ENV === 'development') {
    renderCount++;
    console.log(`ProfiledComponent rendered ${renderCount} times`);
  }
  return <div>Content</div>;
}
```

---

## 17. Common Testing Mistakes

### Testing Implementation Details

❌ Wrong — test breaks on rename even if behavior is unchanged:
```jsx
test('state increments', () => {
  const wrapper = shallow(<Counter />);
  wrapper.instance().setState({ count: 5 });
  expect(wrapper.state('count')).toBe(5);
});
```

✅ Correct — test survives any internal refactoring:
```jsx
test('count increases when increment is clicked', async () => {
  const user = userEvent.setup();
  render(<Counter initialCount={5} />);
  await user.click(screen.getByRole('button', { name: /increment/i }));
  expect(screen.getByText('6')).toBeInTheDocument();
});
```

### Not Clearing Mocks Between Tests

```js
// ❌ Wrong — mock state leaks between tests
jest.mock('./api', () => ({ fetchData: jest.fn() }));

test('first test', () => {
  fetchData.mockReturnValue('data1');
  // ...
});

test('second test — fetchData still has previous mock!', () => {
  // fetchData.mock.calls still contains the call from the first test
});
```

```js
// ✅ Correct — clear mocks in afterEach
afterEach(() => {
  jest.clearAllMocks(); // clears .mock.calls and .mock.instances
});
```

### Ignoring Async Behavior

❌ Wrong — query fires before async update, test may pass for wrong reasons:
```jsx
test('shows data', () => {
  render(<AsyncComponent />);
  expect(screen.getByText('Alice')).toBeInTheDocument(); // may not exist yet
});
```

✅ Correct:
```jsx
test('shows data after fetch', async () => {
  render(<AsyncComponent />);
  expect(await screen.findByText('Alice')).toBeInTheDocument();
});
```

### Over-using Snapshots

❌ Wrong — snapshot of an entire page:
```jsx
test('whole page snapshot', () => {
  const { container } = render(<HomePage />);
  expect(container).toMatchSnapshot(); // 400+ line snapshot nobody reviews
});
```

✅ Correct — behavioral tests for pages, snapshots only for atomic UI:
```jsx
test('home page shows featured products', async () => {
  render(<HomePage />);
  const products = await screen.findAllByRole('article');
  expect(products.length).toBeGreaterThan(0);
});
```

### Using getBy for Asserting Absence

```jsx
// ❌ Wrong — this is an anti-pattern and misleading
expect(() => screen.getByText(/error/i)).toThrow();

// ✅ Correct
expect(screen.queryByText(/error/i)).not.toBeInTheDocument();
```

---

## 18. Best Practices

### Write Tests That Resemble User Behavior

Query what users see (role, label, text). Simulate what users do (click, type, submit). Assert what users observe (text appears, navigation happens).

### Use MSW for API Mocking

MSW is the gold standard for mocking HTTP in React tests. It works at the network level, is reusable across unit/integration/E2E tests, and does not require any changes to the component under test.

### Colocate Test Files with Component Files

```txt
src/
  components/
    SearchBar/
      SearchBar.jsx
      SearchBar.test.jsx      ← colocated
      SearchBar.module.css
    UserCard/
      UserCard.jsx
      UserCard.test.jsx       ← colocated
```

This makes it immediately visible which components lack tests.

### Cover Critical Paths First

Priority order for writing tests:
1. Core user flows (login, checkout, key feature)
2. Error handling paths (failed request, validation failure)
3. Edge cases (empty state, very long input, boundary values)
4. Regression tests: one test per bug that reaches production

### Keep Tests Independent

```jsx
// ❌ Wrong — test depends on previous test's side effect
let sharedState;
test('first', () => { sharedState = 'set'; });
test('second', () => { expect(sharedState).toBe('set'); }); // fragile

// ✅ Correct — each test sets up its own state
test('first', () => { const state = 'set'; expect(state).toBe('set'); });
test('second', () => { const state = 'set'; expect(state).toBe('set'); });
```

### Test Accessibility with jest-axe

```bash
npm install --save-dev jest-axe
```

```jsx
import { axe, toHaveNoViolations } from 'jest-axe';
expect.extend(toHaveNoViolations);

test('form has no accessibility violations', async () => {
  const { container } = render(<LoginForm onSubmit={jest.fn()} />);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

### CI Configuration

```yaml
# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage --watchAll=false --ci
      - uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

### Coverage Thresholds

```js
// jest.config.js
module.exports = {
  coverageThreshold: {
    global: {
      branches: 70,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

Coverage numbers are a safety net, not a goal. 100% coverage does not mean 100% correctness. Focus on testing critical paths thoroughly over gaming coverage metrics.

---
