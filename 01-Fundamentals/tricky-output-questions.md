## 📚 React Fundamentals — Tricky Output Questions

> Questions focusing on core React behavior, JSX, rendering, props, state, and component interaction.

---

## 1. JSX & Rendering

### Q1

```jsx
function App() {
  return <h1>Hello {5 + 5}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
Hello 10
```

JavaScript expressions inside `{}` are evaluated. `5 + 5 = 10` is rendered as text.

</details>

---

### Q2

```jsx
function App() {
  return <h1>{true && "Hello"}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
Hello
```

Logical AND returns the second operand when first is true. `true && "Hello"` evaluates to `"Hello"`.

</details>

---

### Q3

```jsx
function App() {
  return <h1>{false && "Hidden"}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
(empty)
```

React doesn't render boolean values. `false` is filtered out.

</details>

---

### Q4

```jsx
function App() {
  return <h1>{0}</h1>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
0
```

Numbers including `0` are rendered. Only `true`, `false`, `null`, `undefined` are ignored.

</details>

---

### Q5

```jsx
function App() {
  const items = ["apple", "banana"];
  return <div>{items}</div>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
applesbanana
```

Arrays render as concatenated strings without separators. Use `.map()` for proper list rendering.

</details>

---

## 2. Props

### Q6

```jsx
function Child({ name }) {
  name = "Changed";
  return <h1>{name}</h1>;
}

function App() {
  return <Child name="Original" />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
Changed
```

Mutating the prop variable works in JavaScript but violates React's immutability principle. The local reassignment changes the value displayed.

</details>

---

### Q7

```jsx
function Child(props) {
  return <h1>{props.message ?? "Default"}</h1>;
}

function App() {
  return <Child />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
Default
```

Nullish coalescing operator `??` returns the right operand when left is `null` or `undefined`. `props.message` is undefined, so "Default" is used.

</details>

---

### Q8

```jsx
function Child({ count = 0 }) {
  return <h1>{count}</h1>;
}

function App() {
  return <Child count={null} />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
(empty)
```

Default parameters only apply when a prop isn't provided. Passing `null` explicitly means the default `0` is not used. React doesn't render `null`.

</details>

---

## 3. State & Re-rendering

### Q9

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(count + 1);
    console.log(count);
  };

  return <button onClick={increment}>Count: {count}</button>;
}
```

#### ❓ What logs when button is clicked?

<details>
<summary>✅ Answer</summary>

```
0
```

State updates are asynchronous. `console.log(count)` runs before the state actually updates, logging the old value.

</details>

---

### Q10

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  };

  return <h1>{count}</h1>;
}
```

#### ❓ Count after clicking button?

<details>
<summary>✅ Answer</summary>

```
1
```

React batches state updates. All three calls use the same `count` value (0), so all evaluate to `0 + 1 = 1`. Count becomes 1, not 3.

</details>

---

### Q11

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
  };

  return <h1>{count}</h1>;
}
```

#### ❓ Count after clicking button?

<details>
<summary>✅ Answer</summary>

```
3
```

Functional updates receive the result of the previous update. First: 0→1, Second: 1→2, Third: 2→3. Count becomes 3.

</details>

---

## 4. Components & Composition

### Q12

```jsx
function Wrapper(props) {
  return <div>{props.children}</div>;
}

function App() {
  return (
    <Wrapper>
      <h1>Hello</h1>
      <p>World</p>
    </Wrapper>
  );
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<div>
  <h1>Hello</h1>
  <p>World</p>
</div>
```

The `children` prop receives nested elements. Wrapper renders them inside a div.

</details>

---

### Q13

```jsx
const MyComponent = () => <h1>Component</h1>;

function App() {
  const ComponentVar = MyComponent;
  return <ComponentVar />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
Component
```

A component stored in a variable can be rendered because React checks if the tag name is a function.

</details>

---

### Q14

```jsx
function App() {
  const count = 5;
  const DynamicTag = count > 3 ? "h1" : "p";

  return <DynamicTag>Hello</DynamicTag>;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<h1>Hello</h1>
```

You can assign element types to variables. The ternary returns "h1", which React renders as an h1 element.

</details>

---

## 5. Conditional Rendering

### Q15

```jsx
function Alert({ message, show }) {
  return (
    <div>
      {show && <p>{message}</p>}
    </div>
  );
}

function App() {
  return <Alert message="Error!" show={false} />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<div></div>
```

`false && <p>...</p>` evaluates to `false`. React doesn't render the paragraph.

</details>

---

### Q16

```jsx
function Status({ type }) {
  if (type === "loading") return <p>Loading...</p>;
  if (type === "error") return <p>Error</p>;
  return <p>Success</p>;
}

function App() {
  return <Status type="unknown" />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```
Success
```

None of the first two conditions match, so the function returns the final `<p>Success</p>`.

</details>

---

## 6. Lists & Keys

### Q17

```jsx
function List() {
  const items = ["a", "b", "c"];

  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{item}</li>
      ))}
    </ul>
  );
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<ul>
  <li>a</li>
  <li>b</li>
  <li>c</li>
</ul>
```

`.map()` creates elements for each item. Index keys work here for static lists but are problematic for dynamic ones.

</details>

---

### Q18

```jsx
function List() {
  const items = [
    { id: 1, name: "Alice" },
    { id: 2, name: "Bob" }
  ];

  return (
    <ul>
      {items.map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
    </ul>
  );
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<ul>
  <li>Alice</li>
  <li>Bob</li>
</ul>
```

Using stable unique IDs as keys is the best practice.

</details>

---

## 7. Event Handling

### Q19

```jsx
function Button() {
  const handleClick = () => {
    console.log("clicked");
  };

  return <button onClick={handleClick()}>Click</button>;
}
```

#### ❓ What happens?

<details>
<summary>✅ Answer</summary>

```
"clicked" logs immediately during render
Button is broken - clicks don't work
```

`onClick={handleClick()}` immediately invokes the function during render. Use `onClick={handleClick}` without parentheses.

</details>

---

### Q20

```jsx
function Form() {
  const handleSubmit = (e) => {
    e.preventDefault();
    console.log("submitted");
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ Output when form is submitted?

<details>
<summary>✅ Answer</summary>

```
"submitted" logs
Page does not refresh
```

`e.preventDefault()` stops the browser's default form submission (page refresh).

</details>

---

## 8. Advanced Scenarios

### Q21

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
      <Child count={count} />
    </div>
  );
}

function Child({ count }) {
  return <h1>{count}</h1>;
}
```

#### ❓ What happens when button is clicked?

<details>
<summary>✅ Answer</summary>

```
Parent re-renders
Child receives new count prop
Child re-renders with updated value
h1 shows "1"
```

State change in Parent triggers a re-render. Parent passes the new `count` to Child. Child also re-renders because its props changed.

</details>

---

### Q22

```jsx
function App() {
  return (
    <>
      <h1>Hello</h1>
      <p>World</p>
    </>
  );
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<h1>Hello</h1>
<p>World</p>
```

React Fragments (`<>...</>`) allow returning multiple elements without a wrapper div. They don't create extra DOM nodes.

</details>

---

### Q23

```jsx
function App() {
  const element = <h1>Hello</h1>;

  return (
    <div>
      {element}
      {element}
    </div>
  );
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<div>
  <h1>Hello</h1>
  <h1>Hello</h1>
</div>
```

The same element object can be rendered multiple times. React creates separate DOM nodes for each.

</details>

---

### Q24

```jsx
function Child({ render }) {
  return <div>{render()}</div>;
}

function App() {
  return <Child render={() => <h1>Hello</h1>} />;
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

```html
<div>
  <h1>Hello</h1>
</div>
```

Functions can be passed as props and called to render JSX. This is the "render prop" pattern.

</details>

---

### Q25

```jsx
function App() {
  const name = "React";
  const obj = { key: "value" };

  return (
    <div>
      {name}
      {obj}
    </div>
  );
}
```

#### ❓ Output?

<details>
<summary>✅ Answer</summary>

Error:

```
Objects are not valid as a React child
```

Strings render fine. Objects cannot be rendered directly. Extract specific properties: `{obj.key}` or stringify: `{JSON.stringify(obj)}`.

</details>
