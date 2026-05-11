# React Fundamentals

## Table of Contents

- [1. Introduction to React](#1-introduction-to-react)
- [2. Why React is Used](#2-why-react-is-used)
- [3. SPA vs MPA](#3-spa-vs-mpa)
- [4. Declarative vs Imperative Programming](#4-declarative-vs-imperative-programming)
- [5. What is JSX?](#5-what-is-jsx)
- [6. How JSX Works Internally](#6-how-jsx-works-internally)
- [7. Virtual DOM vs Real DOM](#7-virtual-dom-vs-real-dom)
- [8. What are Components?](#8-what-are-components)
- [9. Functional vs Class Components](#9-functional-vs-class-components)
- [10. What are Props?](#10-what-are-props)
- [11. What is State?](#11-what-is-state)
- [12. Event Handling in React](#12-event-handling-in-react)
- [13. Forms in React](#13-forms-in-react)
- [14. Props vs State](#14-props-vs-state)
- [15. React Fragments](#15-react-fragments)
- [16. How React Rendering Works](#16-how-react-rendering-works)
- [17. What is Reconciliation?](#17-what-is-reconciliation)
- [18. React Diffing Algorithm](#18-react-diffing-algorithm)
- [19. Keys in React Lists](#19-keys-in-react-lists)
- [20. React Rendering Flow Summary](#20-react-rendering-flow-summary)
- [21. Best Practices](#21-best-practices)
- [22. React Strict Mode](#22-react-strict-mode)
- [23. React Developer Tools](#23-react-developer-tools)
- [24. Common Beginner Mistakes](#24-common-beginner-mistakes)
- [25. Final Summary](#25-final-summary)

---

## 1. Introduction to React

React is a JavaScript library used for building user interfaces, especially Single Page Applications (SPAs).

It was created by Facebook (Meta) and helps developers build fast, scalable, and reusable frontend applications using components.

---

### Key Features of React

- Component-Based Architecture
- Declarative UI
- Virtual DOM
- Reusable Components
- One-Way Data Flow
- Efficient Rendering
- Strong Ecosystem

---

### Basic Example

```jsx
function App() {
  return <h1>Hello React</h1>;
}
```

---

## 2. Why React is Used

Before React, developers manually updated the DOM using JavaScript, which became difficult in large applications.

React solves many frontend problems by improving:

- Performance
- Maintainability
- Scalability
- Reusability
- Developer Experience

---

### Problems React Solves

#### Manual DOM Manipulation

Traditional JavaScript:

```js
document.getElementById("title").innerText = "Hello";
```

React automatically updates the UI when data changes.

---

#### Reusable UI

```jsx
<Button />
<Button />
<Button />
```

---

#### Efficient Updates

React updates only changed parts of the UI instead of reloading the entire page.

---

## 3. SPA vs MPA

### Single Page Application (SPA)

A Single Page Application loads a single HTML page and dynamically updates content without refreshing the page.

React applications are typically SPAs.

---

### Characteristics of SPA

- Faster navigation
- No full-page reload
- Better user experience
- Client-side rendering

---

### Multi Page Application (MPA)

A Multi Page Application reloads an entirely new page from the server on every navigation.

Traditional websites mostly use MPA architecture.

---

### SPA vs MPA Comparison

| Feature | SPA | MPA |
|---|---|---|
| Page Reload | No | Yes |
| Speed | Faster after initial load | Slower |
| Rendering | Client-side | Server-side |
| User Experience | Smooth | Traditional |
| Navigation | Dynamic | Full reload |

---

## 4. Declarative vs Imperative Programming

### Imperative Programming

Imperative programming focuses on step-by-step instructions.

Example:

```js
const element = document.getElementById("title");

element.innerText = "Hello";
element.style.color = "red";
```

The developer manually controls DOM updates.

---

### Declarative Programming

Declarative programming focuses on describing the desired UI.

Example:

```jsx
<h1 style={{ color: "red" }}>Hello</h1>
```

React handles DOM updates automatically.

---

### Why React Uses Declarative UI

Benefits:

- Cleaner code
- Predictable UI
- Easier maintenance
- Better scalability

---

## 5. What is JSX?

JSX stands for JavaScript XML.

It allows developers to write HTML-like syntax inside JavaScript.

---

### JSX Example

```jsx
const element = <h1>Hello JSX</h1>;
```

Browsers cannot understand JSX directly.

JSX is converted into JavaScript during compilation.

---

### JSX Behind the Scenes

JSX:

```jsx
const element = <h1>Hello</h1>;
```

Converted Version:

```js
const element = React.createElement("h1", null, "Hello");
```

Modern React may compile JSX into:

```js
jsx("h1", { children: "Hello" });
```

---

### JSX Rules

#### JSX Must Return a Single Parent Element

❌ Wrong

```jsx
return (
  <h1>Hello</h1>
  <p>World</p>
);
```

✅ Correct

```jsx
return (
  <>
    <h1>Hello</h1>
    <p>World</p>
  </>
);
```

---

#### Use `className` Instead of `class`

```jsx
<div className="container"></div>
```

---

#### JavaScript Inside JSX Uses Curly Braces

```jsx
const name = "React";

<h1>{name}</h1>;
```

---

#### JSX Prevents XSS Attacks by Default

```jsx
const userInput = "<script>alert('hack')</script>";

<p>{userInput}</p>;
```

React escapes dangerous values automatically.

---

## More JSX Rules

### Self Closing Tags

```jsx
<img src="image.png" />
```

All JSX tags must be properly closed.

---

### JSX Comments

```jsx
{
  /* This is a JSX comment */
}
```

---

### Inline Styles in JSX

```jsx
<div style={{ color: "red", fontSize: "20px" }}>
  Hello
</div>
```

---

### Boolean Values in JSX

```jsx
const isLoggedIn = true;

<h1>{isLoggedIn}</h1>;
```

React does not render boolean values directly.

---

### Null and Undefined Rendering

```jsx
<div>{null}</div>
<div>{undefined}</div>
```

React ignores them during rendering.

---

## Conditional Rendering

Conditional rendering means displaying UI based on conditions.

---

### Using if/else

```jsx
function App() {
  const isLoggedIn = true;

  if (isLoggedIn) {
    return <h1>Welcome</h1>;
  }

  return <h1>Please Login</h1>;
}
```

---

### Using Ternary Operator

```jsx
<h1>{isLoggedIn ? "Welcome" : "Login"}</h1>
```

---

### Using Logical AND Operator

```jsx
{isAdmin && <button>Delete</button>}
```

---

### Early Return Pattern

```jsx
if (!data) return <h1>Loading...</h1>;
```

---

## Rendering Lists

React commonly renders lists using `.map()`.

---

### Example

```jsx
const users = ["A", "B", "C"];

function App() {
  return (
    <ul>
      {users.map((user) => (
        <li key={user}>{user}</li>
      ))}
    </ul>
  );
}
```

---

## 6. How JSX Works Internally

### JSX Compilation Flow

```text
JSX
 ↓
Babel Transpilation
 ↓
React.createElement()
 ↓
JavaScript Object
 ↓
Virtual DOM
 ↓
Real DOM
```

---

### React Element Object

```jsx
const element = <h1>Hello</h1>;
```

Converted object:

```js
{
  type: "h1",
  props: {
    children: "Hello"
  }
}
```

React elements are plain JavaScript objects.

---

## 7. Virtual DOM vs Real DOM

## Real DOM

The Real DOM is the actual browser DOM.

Updating the Real DOM directly is expensive because the browser must recalculate layout and repaint UI.

---

## Virtual DOM

The Virtual DOM is a lightweight JavaScript representation of the Real DOM.

React first updates the Virtual DOM and then efficiently updates only necessary parts of the Real DOM.

---

### Virtual DOM Update Process

```text
State Change
     ↓
New Virtual DOM Created
     ↓
Compare with Previous Virtual DOM
     ↓
Find Differences
     ↓
Update Real DOM Efficiently
```

---

### Example

Initial UI:

```jsx
<h1>Hello</h1>
```

Updated UI:

```jsx
<h1>Hello World</h1>
```

React updates only the changed text node.

---

### Advantages of Virtual DOM

- Faster updates
- Better performance
- Reduced DOM operations
- Efficient rendering

---

## 8. What are Components?

Components are reusable building blocks in React.

Each component returns a piece of UI.

---

## Component-Based Architecture

React applications are built using independent reusable components.

---

### Advantages

- Reusability
- Better organization
- Easier debugging
- Better scalability
- Independent logic handling

---

### Functional Component

```jsx
function Welcome() {
  return <h1>Hello</h1>;
}
```

---

## Component Composition

Large UIs can be broken into smaller reusable components.

```jsx
<App>
  <Navbar />
  <Sidebar />
  <Dashboard />
</App>
```

---

### Benefits of Components

- Reusability
- Better code organization
- Easier maintenance
- Independent logic

---

## 9. Functional vs Class Components

## Functional Components

```jsx
function App() {
  return <h1>Hello</h1>;
}
```

---

### Features

- Simpler syntax
- Hooks support
- Better readability
- Less boilerplate

---

## Class Components

```jsx
class App extends React.Component {
  render() {
    return <h1>Hello</h1>;
  }
}
```

---

### Differences

| Feature | Functional | Class |
|---|---|---|
| Syntax | Simple | Complex |
| State | Hooks | `this.state` |
| Lifecycle | Hooks | Lifecycle Methods |
| `this` keyword | Not used | Used |
| Modern Usage | Preferred | Legacy |

---

## 10. What are Props?

Props (Properties) are used to pass data from parent to child components.

Props are read-only.

---

### Example

```jsx
function User(props) {
  return <h1>{props.name}</h1>;
}

function App() {
  return <User name="Dilkhush" />;
}
```

---

### Props Flow

```text
Parent Component
      ↓
     Props
      ↓
Child Component
```

---

### Props are Immutable

❌ Wrong

```jsx
props.name = "New";
```

Props should never be modified.

---

## Default Props

Default props provide fallback values.

```jsx
function Button({ text = "Click" }) {
  return <button>{text}</button>;
}
```

---

## Children Props

Components can receive nested elements using `children`.

```jsx
function Card({ children }) {
  return <div>{children}</div>;
}
```

Usage:

```jsx
<Card>
  <h1>Hello</h1>
</Card>
```

---

## Callback Props

Functions can also be passed as props.

```jsx
function Child({ handleClick }) {
  return <button onClick={handleClick}>Click</button>;
}
```

---

## Props Drilling

Passing props through multiple components is called props drilling.

```text
App → Parent → Child → GrandChild
```

Large applications often solve this using Context API or state management libraries.

---

## 11. What is State?

State is data managed inside a component.

When state changes, React re-renders the component.

---

### Example

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

---

### Important State Rules

#### Never Mutate State Directly

❌ Wrong

```js
count = count + 1;
```

✅ Correct

```js
setCount(count + 1);
```

---

#### State Updates Trigger Re-render

React automatically updates the UI after state changes.

---

#### State is Local

Each component has its own independent state.

---

## Why State Updates are Async

React batches state updates for performance optimization.

```jsx
setCount(count + 1);
setCount(count + 1);
```

Updates may not happen immediately.

---

## Functional Updates

```jsx
setCount((prev) => prev + 1);
```

Useful when new state depends on previous state.

---

## Immutability in React

React relies on reference comparison.

---

### Wrong Way

```js
user.name = "React";
```

---

### Correct Way

```jsx
setUser({
  ...user,
  name: "React"
});
```

---

## 12. Event Handling in React

React handles events using camelCase syntax.

---

### Example

```jsx
<button onClick={handleClick}>
  Click
</button>
```

---

### Event Object

```jsx
function handleClick(event) {
  console.log(event);
}
```

---

### preventDefault()

```jsx
function handleSubmit(e) {
  e.preventDefault();
}
```

Prevents default browser behavior.

---

### stopPropagation()

```jsx
function handleClick(e) {
  e.stopPropagation();
}
```

Stops event bubbling.

---

## Synthetic Events

React wraps browser events inside SyntheticEvent objects for cross-browser consistency.

---

## 13. Forms in React

## Controlled Components

React controls form data using state.

```jsx
function App() {
  const [name, setName] = useState("");

  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  );
}
```

---

## Uncontrolled Components

DOM handles form state directly.

```jsx
<input ref={inputRef} />
```

---

## 14. Props vs State

| Feature | Props | State |
|---|---|---|
| Owned By | Parent | Component |
| Mutable | No | Yes |
| Purpose | Pass Data | Manage Data |
| Re-render Trigger | Parent Updates | State Updates |
| Accessibility | Child Receives | Local Component |

---

## 15. React Fragments

Fragments allow grouping multiple elements without adding extra DOM nodes.

---

### Without Fragment

```jsx
<div>
  <h1>Hello</h1>
  <p>World</p>
</div>
```

---

### With Fragment

```jsx
<>
  <h1>Hello</h1>
  <p>World</p>
</>
```

---

### Full Syntax

```jsx
<React.Fragment>
  <h1>Hello</h1>
</React.Fragment>
```

---

## 16. How React Rendering Works

Rendering means converting React components into UI elements.

---

### Rendering Flow

```text
State/Props Change
        ↓
Component Re-renders
        ↓
New Virtual DOM Created
        ↓
Diffing Happens
        ↓
DOM Updated Efficiently
```

---

### Important Point

React does not reload the full page.

Only required UI parts are updated.

---

### Re-render Triggers

- State changes
- Props changes
- Parent re-render
- Context updates

---

## React Rendering Phases

## Render Phase

React creates Virtual DOM and calculates UI changes.

---

## Commit Phase

React updates the Real DOM.

---

## Parent and Child Re-rendering

When a parent component re-renders:

- Child components also re-render by default
- Even if props are unchanged

---

## 17. What is Reconciliation?

Reconciliation is React’s process of comparing old and new Virtual DOM trees.

React determines the minimum DOM changes required.

---

### Example

Old UI:

```jsx
<h1>Hello</h1>
```

New UI:

```jsx
<h1>Hello React</h1>
```

React updates only the text content.

---

### Goals of Reconciliation

- Minimize DOM operations
- Improve performance
- Efficient UI updates

---

## 18. React Diffing Algorithm

React uses an optimized diffing algorithm to compare Virtual DOM trees efficiently.

---

### React Assumptions

#### Different Element Types Produce Different Trees

```jsx
<div />
```

to

```jsx
<span />
```

React destroys the old tree and creates a new one.

---

#### Keys Help Identify Elements

Keys help React track list items during updates.

---

### Time Complexity

Traditional deep comparison:

```text
O(n³)
```

React optimized diffing:

```text
O(n)
```

---

## 19. Keys in React Lists

Keys are unique identifiers used in lists.

---

### Example

```jsx
const users = ["A", "B", "C"];

users.map((user, index) => (
  <li key={index}>{user}</li>
));
```

---

### Why Keys Are Important

Keys help React:

- Identify elements
- Reuse DOM nodes
- Optimize rendering
- Prevent unnecessary re-renders

---

### Best Practices

✅ Use stable unique IDs

```jsx
key={user.id}
```

❌ Avoid random keys

```jsx
key={Math.random()}
```

❌ Avoid index keys in dynamic lists

```jsx
key={index}
```

---

### Problems with Index Keys

- Incorrect UI updates
- State mismatch
- Performance issues

---

## 20. React Rendering Flow Summary

```text
Component Render
       ↓
JSX Returned
       ↓
React Element Created
       ↓
Virtual DOM Generated
       ↓
Diffing & Reconciliation
       ↓
Minimal DOM Updates
       ↓
UI Updated
```

---

## 21. Best Practices

### Keep Components Small

Smaller components are easier to reuse and maintain.

---

### Prefer Functional Components

Modern React primarily uses hooks-based functional components.

---

### Avoid Direct DOM Manipulation

React should control the UI updates.

---

### Use Proper Keys

Always use stable and unique keys.

---

### Never Mutate State

Always create new state values instead of modifying existing ones.

---

### Keep UI Declarative

Describe UI based on current state and props.

---

## 22. React Strict Mode

StrictMode helps detect potential issues during development.

```jsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

---

### Important Notes

- Development-only feature
- May intentionally double invoke some functions
- Helps identify unsafe side effects

---

## 23. React Developer Tools

React Developer Tools browser extension helps developers:

- Inspect components
- View props/state
- Debug rendering
- Analyze component tree

---

## 24. Common Beginner Mistakes

### Mutating State Directly

❌ Wrong

```js
user.name = "New";
```

---

### Using Index as Key Everywhere

Can create rendering bugs in dynamic lists.

---

### Forgetting Single Parent Element

JSX must return one parent wrapper.

---

### Confusing Props and State

- Props → External Data
- State → Internal Data

---

### Writing Too Much Logic Inside JSX

Keep JSX clean and readable.

---

## Infinite Re-renders

❌ Wrong

```jsx
<button onClick={setCount(count + 1)}>
```

This executes immediately during render.

---

✅ Correct

```jsx
<button onClick={() => setCount(count + 1)}>
```

---

## Direct DOM Manipulation

❌ Avoid:

```js
document.getElementById("title");
```

React should control the UI whenever possible.

---

## 25. Final Summary

React is a modern JavaScript library for building fast and scalable user interfaces.

Core React concepts include:

- JSX
- Components
- Props
- State
- Virtual DOM
- Rendering
- Reconciliation
- Diffing
- Keys
- Event Handling
- Forms
- Declarative UI
- Component Architecture

Understanding these fundamentals is essential before learning advanced topics like:

- Hooks
- Context API
- Routing
- State Management
- Performance Optimization
- Server Components
- React Architecture