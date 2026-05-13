## 📚 React Testing and Debugging — Tricky Output Questions

> These questions test deep understanding of React Testing Library behavior, Jest mocking internals, async test patterns, and common bugs. Each question presents a code snippet followed by a predicted output or behavior question. Work through each before revealing the answer.

---

## 1. Queries

### Q1

```jsx
import { render, screen } from '@testing-library/react';

function Message() {
  return null;
}

test('query behavior', () => {
  render(<Message />);
  const el = screen.getByText('Hello');
  console.log(el);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
TestingLibraryElementError: Unable to find an element with the text: Hello.
```

`getByText` throws immediately if the element is not found. Since `<Message />` renders `null`, there is no DOM node with the text "Hello". The test fails with a descriptive error. To safely check for absence, use `queryByText` which returns `null` instead of throwing.

</details>

---

### Q2

```jsx
import { render, screen } from '@testing-library/react';

function Message() {
  return null;
}

test('query behavior', () => {
  render(<Message />);
  const el = screen.queryByText('Hello');
  console.log(el);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
null
```

`queryByText` returns `null` when the element is not found — it does NOT throw. The `console.log` prints `null`. The test passes because there is no failing assertion. Use `queryBy` whenever you want to assert an element is absent: `expect(screen.queryByText('Hello')).not.toBeInTheDocument()`.

</details>

---

### Q3

```jsx
import { render, screen } from '@testing-library/react';

function Buttons() {
  return (
    <div>
      <button>Save</button>
      <button>Cancel</button>
    </div>
  );
}

test('query multiple', () => {
  render(<Buttons />);
  const btn = screen.getByRole('button');
  console.log(btn.textContent);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
TestingLibraryElementError: Found multiple elements with the role "button"
```

`getByRole` (and all singular `getBy*` queries) throw when more than one element matches. Since there are two `<button>` elements, the query throws. The fix is either to use `getAllByRole('button')` to get both, or to narrow the query: `getByRole('button', { name: /save/i })`.

</details>

---

### Q4

```jsx
import { render, screen } from '@testing-library/react';

function AsyncComponent() {
  const [loaded, setLoaded] = React.useState(false);

  React.useEffect(() => {
    setTimeout(() => setLoaded(true), 200);
  }, []);

  return <div>{loaded ? 'Done' : 'Loading'}</div>;
}

test('async query', async () => {
  render(<AsyncComponent />);
  const el = await screen.findByText('Done');
  console.log(el.textContent);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
Done
```

`findByText` is an async query that polls every 50ms until the text appears or the default 1000ms timeout expires. After 200ms the `setTimeout` fires, state updates to `true`, the component re-renders with "Done", and `findByText` resolves with the DOM node. `el.textContent` is `'Done'`. The test passes.

</details>

---

## 2. User Events

### Q5

```jsx
import userEvent from '@testing-library/user-event';
import { render, screen } from '@testing-library/react';

test('type into input', async () => {
  const user = userEvent.setup();
  render(<input type="text" />);

  const input = screen.getByRole('textbox');
  await user.type(input, 'abc');

  console.log(input.value);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
abc
```

`userEvent.type` simulates character-by-character keydown → keypress → input → keyup events for each character. After typing 'a', 'b', 'c', the input's value is `'abc'`. This is an uncontrolled input so the DOM value updates directly from the simulated events.

</details>

---

### Q6

```jsx
import userEvent from '@testing-library/user-event';
import { render, screen } from '@testing-library/react';

test('click counter', async () => {
  const handler = jest.fn();
  const user = userEvent.setup();

  render(<button onClick={handler}>Click</button>);

  await user.click(screen.getByRole('button'));
  await user.click(screen.getByRole('button'));

  console.log(handler.mock.calls.length);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
2
```

Each `user.click()` fires a full pointer + click event sequence. The handler is attached to the button's `onClick` and is called once per click. After two clicks, `handler.mock.calls.length` is `2`. Note: `jest.fn()` records all calls in `.mock.calls`, an array of call argument arrays.

</details>

---

### Q7

```jsx
import userEvent from '@testing-library/user-event';
import { render, screen } from '@testing-library/react';

function ControlledInput() {
  const [val, setVal] = React.useState('');
  return (
    <input
      type="text"
      value={val}
      onChange={e => setVal(e.target.value)}
      aria-label="Name"
    />
  );
}

test('controlled input type', async () => {
  const user = userEvent.setup();
  render(<ControlledInput />);

  const input = screen.getByLabelText('Name');
  await user.type(input, 'React');

  console.log(input.value);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
React
```

For a controlled input, each character typed fires an `onChange` event. The event handler calls `setVal(e.target.value)`, which updates React state, which triggers re-render with the new value prop. RTL correctly simulates this cycle for each character. After typing all 5 characters, `input.value` is `'React'`.

</details>

---

### Q8

```jsx
import userEvent from '@testing-library/user-event';
import { render, screen } from '@testing-library/react';

function Form({ onSubmit }) {
  return (
    <form onSubmit={e => { e.preventDefault(); onSubmit(); }}>
      <button type="submit">Submit</button>
    </form>
  );
}

test('form submit', async () => {
  const handleSubmit = jest.fn();
  const user = userEvent.setup();

  render(<Form onSubmit={handleSubmit} />);
  await user.click(screen.getByRole('button', { name: /submit/i }));

  console.log(handleSubmit.mock.calls.length);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
1
```

Clicking a `<button type="submit">` inside a `<form>` triggers the form's `submit` event. The handler calls `e.preventDefault()` (preventing actual navigation) and then calls `onSubmit()`. `handleSubmit` is called once. `jest.fn()` records this in `mock.calls`, giving a length of `1`.

</details>

---

## 3. Async Testing

### Q9

```jsx
import { render, screen, waitFor } from '@testing-library/react';

function Delayed() {
  const [show, setShow] = React.useState(false);

  React.useEffect(() => {
    const id = setTimeout(() => setShow(true), 500);
    return () => clearTimeout(id);
  }, []);

  return <div>{show ? 'Visible' : ''}</div>;
}

test('waitFor resolves', async () => {
  render(<Delayed />);

  await waitFor(() => {
    expect(screen.getByText('Visible')).toBeInTheDocument();
  }, { timeout: 1000 });

  console.log('test passed');
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
test passed
```

`waitFor` calls its callback on an interval (default 50ms) until it stops throwing. The component renders an empty string initially. After 500ms the `setTimeout` fires, state becomes `true`, re-render produces "Visible" in the DOM. On the next `waitFor` poll after 500ms, `getByText('Visible')` succeeds. The `timeout: 1000` gives enough headroom. The test passes and logs `'test passed'`.

</details>

---

### Q10

```jsx
import { render, screen } from '@testing-library/react';

function NeverLoads() {
  return <p>Loading...</p>;
}

test('findBy timeout', async () => {
  render(<NeverLoads />);

  try {
    await screen.findByText('Done', {}, { timeout: 100 });
  } catch (e) {
    console.log(e.name);
  }
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
TestingLibraryElementError
```

`findByText` waits up to the specified `timeout` (100ms here). Since the component never renders "Done", the query times out and throws a `TestingLibraryElementError`. The `catch` block logs the error's `name` property, which is `'TestingLibraryElementError'`. The test itself passes because the error is caught.

</details>

---

### Q11

```jsx
import { render, screen, waitFor } from '@testing-library/react';

test('multiple assertions in waitFor', async () => {
  render(<div><span>A</span><span>B</span></div>);

  await waitFor(() => {
    expect(screen.getByText('A')).toBeInTheDocument();
    expect(screen.getByText('C')).toBeInTheDocument(); // C does not exist
  });
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
TestingLibraryElementError: Unable to find an element with the text: C
(test fails after timeout)
```

`waitFor` keeps retrying the entire callback until all assertions inside pass or the timeout expires. Since "C" never appears in the DOM, the second assertion always throws, and `waitFor` eventually times out and throws. A key insight: put only ONE assertion inside `waitFor`. After `waitFor` resolves, add additional assertions outside it. Multiple assertions inside `waitFor` can hide which one is actually failing.

</details>

---

### Q12

```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

function Toggler() {
  const [count, setCount] = React.useState(0);

  async function handleClick() {
    await new Promise(resolve => setTimeout(resolve, 100));
    setCount(c => c + 1);
  }

  return <button onClick={handleClick}>{count}</button>;
}

test('async state update after click', async () => {
  const user = userEvent.setup();
  render(<Toggler />);

  await user.click(screen.getByRole('button'));
  expect(await screen.findByText('1')).toBeInTheDocument();
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
(test passes)
```

After `user.click`, the `handleClick` handler runs asynchronously. After 100ms the `setTimeout` resolves, state updates to `1`, and the button re-renders with `'1'`. `findByText('1')` waits up to 1000ms and resolves as soon as the text appears. The test passes. Note: if you used `getByText('1')` synchronously right after click, it would fail because the state update is delayed.

</details>

---

## 4. Mocking

### Q13

```js
const fn = jest.fn()
  .mockReturnValueOnce('first')
  .mockReturnValueOnce('second')
  .mockReturnValue('default');

console.log(fn());
console.log(fn());
console.log(fn());
console.log(fn());
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
first
second
default
default
```

`mockReturnValueOnce` values are consumed in order, one per call. Once the `Once` values are exhausted, the `mockReturnValue` fallback takes over for all subsequent calls. First call → `'first'`, second call → `'second'`, third and fourth calls → `'default'`.

</details>

---

### Q14

```js
jest.mock('./math', () => ({
  add: jest.fn().mockReturnValue(10),
}));

import { add } from './math';

test('mock module', () => {
  const result = add(2, 3);
  console.log(result);
  console.log(add.mock.calls[0]);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
10
[2, 3]
```

`jest.mock` replaces the `./math` module entirely. `add` is now a `jest.fn()` that always returns `10` regardless of arguments. Calling `add(2, 3)` returns `10`. The call is recorded in `add.mock.calls[0]`, which is `[2, 3]` — the array of arguments from the first call.

</details>

---

### Q15

```js
const mockFetch = jest.fn().mockResolvedValue({
  ok: true,
  json: jest.fn().mockResolvedValue({ name: 'Alice' }),
});

global.fetch = mockFetch;

test('fetch mock', async () => {
  const res = await fetch('/api/user');
  const data = await res.json();
  console.log(data.name);
  console.log(fetch.mock.calls.length);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
Alice
1
```

`global.fetch` is replaced with `mockFetch`. Calling `fetch('/api/user')` resolves to the mock response object `{ ok: true, json: mockFn }`. Calling `.json()` on it resolves to `{ name: 'Alice' }`. `data.name` is `'Alice'`. `fetch.mock.calls.length` is `1` because fetch was called exactly once.

</details>

---

### Q16

```js
const fn = jest.fn();

fn(1);
fn(2);
fn(3);

jest.clearAllMocks();

console.log(fn.mock.calls.length);
fn(4);
console.log(fn.mock.calls.length);
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
0
1
```

`jest.clearAllMocks()` resets `.mock.calls`, `.mock.instances`, and `.mock.results` to empty arrays — it clears all recorded call history. After clearing, `fn.mock.calls.length` is `0`. The next call `fn(4)` is recorded as the first call, making length `1`. Note: `clearAllMocks` does NOT remove mock implementations set with `mockReturnValue` — that requires `jest.resetAllMocks()`.

</details>

---

## 5. Custom Hooks

### Q17

```jsx
import { renderHook, act } from '@testing-library/react';

function useToggle(initial = false) {
  const [state, setState] = React.useState(initial);
  const toggle = () => setState(s => !s);
  return [state, toggle];
}

test('useToggle hook', () => {
  const { result } = renderHook(() => useToggle(false));

  console.log(result.current[0]);

  act(() => {
    result.current[1](); // toggle
  });

  console.log(result.current[0]);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
false
true
```

`renderHook` mounts the hook and exposes its return value via `result.current`. The initial state is `false`, so `result.current[0]` logs `false`. Inside `act`, calling `result.current[1]()` (the toggle function) sets state to `!false = true`. After `act` flushes, `result.current` is updated and `result.current[0]` is now `true`.

</details>

---

### Q18

```jsx
import { renderHook, act } from '@testing-library/react';

function useCounter(initial = 0) {
  const [count, setCount] = React.useState(initial);
  const increment = () => setCount(c => c + 1);
  return { count, increment };
}

test('multiple increments', () => {
  const { result } = renderHook(() => useCounter(0));

  act(() => {
    result.current.increment();
    result.current.increment();
    result.current.increment();
  });

  console.log(result.current.count);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
3
```

All three `increment()` calls are batched inside a single `act()`. React batches state updates within `act`, so the functional updater `c => c + 1` is applied three times: 0 → 1 → 2 → 3. After `act` completes, `result.current.count` reflects the final batched state of `3`.

</details>

---

### Q19

```jsx
import { renderHook } from '@testing-library/react';

function useCounter() {
  const [count, setCount] = React.useState(0);
  const increment = () => setCount(c => c + 1);
  return { count, increment };
}

test('increment without act', () => {
  const { result } = renderHook(() => useCounter());
  result.current.increment(); // no act wrapper
  console.log(result.current.count);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
0
(plus React warning: "An update to ... inside a test was not wrapped in act(...)")
```

Without wrapping the `increment()` call in `act()`, React state updates are not flushed synchronously in the test environment. `result.current.count` is still `0` when logged. React also emits a warning about the unhandled state update. Always wrap calls that trigger state updates in `act()` when using `renderHook`.

</details>

---

## 6. Common Bugs

### Q20

```jsx
import { render, screen } from '@testing-library/react';

function Timer() {
  const [sec, setSec] = React.useState(0);

  React.useEffect(() => {
    const id = setInterval(() => setSec(s => s + 1), 1000);
    // no cleanup
  }, []);

  return <p>{sec}</p>;
}

test('timer component', () => {
  render(<Timer />);
  expect(screen.getByText('0')).toBeInTheDocument();
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
(test passes but React emits a warning)

Warning: An update to Timer inside a test was not wrapped in act(...).
```

The test itself passes because `getByText('0')` finds the initial rendered value. However, the `setInterval` is still running after the test completes (no cleanup function returned from `useEffect`). After the test, RTL's `cleanup` unmounts the component, but the interval continues calling `setSec`, which tries to update state on an unmounted component. React warns about this. Always return a cleanup from `useEffect` when setting up intervals.

</details>

---

### Q21

```jsx
test('assertion order matters', async () => {
  render(<div id="root"><span>Hello</span></div>);

  // Synchronous query right after render
  const span = screen.getByText('Hello');
  expect(span).toBeInTheDocument();

  // Component is unmounted by cleanup
});

test('second test', () => {
  // screen has been cleaned up between tests
  console.log(screen.queryByText('Hello'));
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
null
```

RTL registers `cleanup` with `afterEach` automatically. After the first test, the rendered component is unmounted and the jsdom DOM is cleared. In the second test, `screen.queryByText('Hello')` returns `null` because the DOM no longer contains that element. This demonstrates that RTL tests are correctly isolated — state does not leak between tests.

</details>

---

### Q22

```jsx
import { render, screen } from '@testing-library/react';

function BadComponent() {
  const [val, setVal] = React.useState(0);
  setVal(1); // called unconditionally in render body
  return <div>{val}</div>;
}

test('infinite render bug', () => {
  expect(() => render(<BadComponent />)).toThrow();
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
(test passes — render throws "Too many re-renders")
```

Calling a state setter (`setVal`) unconditionally during the render body creates an infinite re-render loop: render → state update → re-render → state update → ... React detects this after a threshold (currently 25 re-renders) and throws "Too many re-renders. React limits the number of renders to prevent an infinite loop." The `expect(() => render(...)).toThrow()` assertion catches this error and the test passes.

</details>

---

## 7. Edge Cases

### Q23

```jsx
import { render } from '@testing-library/react';
import Button from './Button';

test('snapshot matches', () => {
  const { container } = render(<Button variant="primary">Save</Button>);
  expect(container).toMatchSnapshot();
});
```

The snapshot file already contains:
```txt
exports[`snapshot matches 1`] = `
<div>
  <button class="btn btn-secondary">Save</button>
</div>
`;
```

But `Button.jsx` currently outputs `class="btn btn-primary"`. What happens?

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
Snapshot name: `snapshot matches 1`

- Snapshot  - 1
+ Received  + 1

    <div>
      <button
-       class="btn btn-secondary"
+       class="btn btn-primary"
      >
        Save
      </button>
    </div>

Snapshot Summary: 1 snapshot failed.
```

Jest diffs the current render against the stored snapshot and finds a mismatch in the `class` attribute. The test fails and prints a colored diff. If the change to `btn-primary` is intentional, run `jest --updateSnapshot` to update the `.snap` file. This is the primary value of snapshot testing: detecting unintentional changes.

</details>

---

### Q24

```jsx
import { render, screen } from '@testing-library/react';

function List({ items }) {
  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}

test('rerender updates DOM', () => {
  const { rerender } = render(
    <List items={[{ id: 1, name: 'Apple' }]} />
  );

  expect(screen.getAllByRole('listitem')).toHaveLength(1);

  rerender(
    <List items={[{ id: 1, name: 'Apple' }, { id: 2, name: 'Banana' }]} />
  );

  console.log(screen.getAllByRole('listitem').length);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
2
```

`rerender` re-renders the same component with new props, in the same container. React diffs the previous render (1 `<li>`) with the new render (2 `<li>` elements) and applies the minimal DOM update. After `rerender`, the DOM contains two list items. `getAllByRole('listitem').length` returns `2`. This is equivalent to a parent component re-rendering with new data.

</details>

---

### Q25

```jsx
import { render, screen } from '@testing-library/react';
import { createContext, useContext } from 'react';

const CountContext = createContext(0);

function Display() {
  const count = useContext(CountContext);
  return <span data-testid="count">{count}</span>;
}

test('context value in test', () => {
  render(
    <CountContext.Provider value={42}>
      <Display />
    </CountContext.Provider>
  );

  console.log(screen.getByTestId('count').textContent);
});
```

#### ❓ Output?

<details><summary>✅ Answer</summary>

```txt
42
```

When a component is rendered inside a `<Context.Provider>` in a test, it receives the provided value exactly as it would in a real application. `useContext(CountContext)` returns `42` (the value prop), and the `<span>` renders `'42'`. `textContent` is the string `'42'`. This is the correct pattern for testing context-consuming components without needing the real provider logic.

</details>

---

## ✅ Topics Covered

- `getBy` throws when element is absent
- `queryBy` returns null instead of throwing
- `getBy` throws when multiple elements match
- `findBy` async query with polling until timeout
- `userEvent.type` for uncontrolled and controlled inputs
- `userEvent.click` call count tracking via `mock.calls.length`
- Async state update requiring `findBy` after user action
- Form submit via `<button type="submit">`
- `waitFor` retry behavior until assertion passes
- `findBy` timeout resulting in `TestingLibraryElementError`
- Multiple assertions inside `waitFor` and why to avoid it
- Async state update patterns with `findBy`
- `mockReturnValueOnce` consumption order and fallback
- `jest.mock` replacing module with custom implementation
- `global.fetch` mocking and `mock.calls` inspection
- `jest.clearAllMocks` clearing call history without removing implementation
- `renderHook` result.current access for hook return values
- Multiple state updates batched inside single `act()`
- State update without `act()` — stale result and React warning
- Missing `useEffect` cleanup causing state update on unmounted component
- RTL automatic cleanup between tests
- Infinite re-render from setState in render body
- Snapshot mismatch detection and diff output
- `rerender` updating DOM with new props
- Context value injection in tests via Provider wrapper
