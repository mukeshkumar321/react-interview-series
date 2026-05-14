# React Fundamentals

## Table of Contents

- [1. What is React?](#1-what-is-react)
- [2. Core Concepts](#2-core-concepts)
- [3. JSX Basics](#3-jsx-basics)
- [4. Components](#4-components)
- [5. Props](#5-props)
- [6. State](#6-state)
- [7. Virtual DOM](#7-virtual-dom)
- [8. Rendering](#8-rendering)
- [9. Event Handling](#9-event-handling)
- [10. Forms & Input Handling](#10-forms--input-handling)
- [11. Conditional Rendering](#11-conditional-rendering)
- [12. Lists & Keys](#12-lists--keys)
- [13. Component Lifecycle](#13-component-lifecycle)
- [14. Best Practices](#14-best-practices)

---

## 1. What is React?

React is a JavaScript library created by Meta (formerly Facebook) for building user interfaces with reusable components.

It provides a declarative way to describe what the UI should look like based on the current state and props.

---

### Why React?

**Problems it solves:**

- Manual DOM manipulation is error-prone and difficult to maintain
- Building interactive UIs requires managing complex state and side effects
- Reusable components reduce code duplication
- Efficient rendering prevents unnecessary DOM updates

**Key advantages:**

- Component-based architecture
- Declarative UI description
- Virtual DOM for efficient updates
- Strong ecosystem and community
- Easy to learn and master

---

## 2. Core Concepts

### Single Page Application (SPA)

A Single Page Application loads a single HTML file and dynamically updates content without full-page reloads.

React applications are typically SPAs.

---

### Declarative vs Imperative

**Imperative (HOW):**

Step-by-step instructions to manipulate the DOM directly.

**Declarative (WHAT):**

Describe the desired UI state, and React handles the updates.

---

### Component-Based Architecture

React applications are composed of small, reusable components that manage their own logic and rendering.

---

## 3. JSX Basics

JSX is a syntax extension for JavaScript that allows you to write HTML-like code inside JavaScript.

---

### JSX Compilation

JSX is transpiled to `React.createElement()` calls during build time.

```jsx
// JSX
<h1 className="title">Hello</h1>

// Transpiled
React.createElement("h1", { className: "title" }, "Hello")
```

---

### JSX Rules

**Single root element:**

A component must return one parent element or a Fragment.

**className instead of class:**

```jsx
<div className="container">Content</div>
```

**JavaScript expressions in curly braces:**

```jsx
const name = "React";
<h1>Hello {name}</h1>
```

**Self-closing tags:**

```jsx
<img src="image.png" />
<input type="text" />
```

---

### Special Values in JSX

React does not render:
- `true`
- `false`
- `null`
- `undefined`

But numbers including `0` are rendered.

---

## 4. Components

Components are reusable building blocks that return a piece of UI.

---

### Functional Components

```jsx
function Welcome() {
  return <h1>Hello!</h1>;
}
```

Functional components are the modern preferred approach.

---

### Component Naming

- Component names must start with a capital letter
- React treats lowercase as DOM elements

```jsx
<Welcome />   // Custom component
<div />       // DOM element
```

---

### Component Composition

Break complex UIs into smaller components:

```jsx
function App() {
  return (
    <div>
      <Header />
      <Body />
      <Footer />
    </div>
  );
}
```

---

### Children Props

Components can receive nested content:

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}

<Card>
  <h1>Title</h1>
  <p>Content</p>
</Card>
```

---

## 5. Props

Props (Properties) are read-only data passed from parent to child components.

---

### Basic Props

```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

<Greeting name="Alice" />
```

---

### Default Props

```jsx
function Button({ text = "Click me" }) {
  return <button>{text}</button>;
}
```

---

### Props Spreading

```jsx
function Input({ label, ...inputProps }) {
  return (
    <div>
      <label>{label}</label>
      <input {...inputProps} />
    </div>
  );
}

<Input label="Email" type="email" placeholder="Enter" />
```

---

### Props Are Immutable

Never modify props directly. They should be treated as read-only.

---

### Props Drilling

Passing props through multiple component levels can become cumbersome. Solutions include Context API or state management libraries.

---

## 6. State

State is data managed internally by a component that can change over time.

When state changes, React re-renders the component.

---

### useState Hook

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      Count: {count}
    </button>
  );
}
```

---

### State Rules

**1. Never mutate state directly:**

```jsx
// Wrong
state.name = "new";

// Correct
setState({ ...state, name: "new" });
```

**2. State updates are asynchronous:**

Changes don't apply immediately.

**3. State updates are batched:**

Multiple updates in an event handler are combined into one re-render.

---

### Functional Updates

When new state depends on previous state:

```jsx
setCount(prev => prev + 1);
```

---

### State with Objects and Arrays

Always create new objects/arrays when updating:

```jsx
// Objects
setUser({ ...user, name: "New" });

// Arrays
setItems([...items, newItem]);
```

---

## 7. Virtual DOM

The Virtual DOM is a lightweight JavaScript representation of the actual browser DOM.

---

### How It Works

1. State/props change
2. New Virtual DOM is created
3. React compares old and new Virtual DOM
4. Only changed elements update the Real DOM
5. UI is updated

---

### Benefits

- Faster updates than direct DOM manipulation
- Efficient rendering
- Reduced DOM operations

---

## 8. Rendering

Rendering is the process of converting React components into DOM elements.

---

### When Components Re-render

- State changes
- Props change
- Parent component re-renders
- Context value changes

---

### Parent-Child Re-rendering

When a parent re-renders, children also re-render by default, even if their props haven't changed.

Use `React.memo` to prevent unnecessary child re-renders.

---

## 9. Event Handling

React uses camelCase event names (onClick, onChange, onSubmit, etc.).

---

### Basic Event Handling

```jsx
function Button() {
  const handleClick = () => {
    console.log("clicked");
  };

  return <button onClick={handleClick}>Click</button>;
}
```

---

### Event Object

```jsx
function Input() {
  const handleChange = (e) => {
    console.log(e.target.value);
  };

  return <input onChange={handleChange} />;
}
```

---

### Preventing Default Behavior

```jsx
function Form() {
  const handleSubmit = (e) => {
    e.preventDefault();
    // Handle form submission
  };

  return <form onSubmit={handleSubmit}>...</form>;
}
```

---

## 10. Forms & Input Handling

### Controlled Components

React controls form state through component state:

```jsx
function Form() {
  const [input, setInput] = useState("");

  return (
    <input
      value={input}
      onChange={(e) => setInput(e.target.value)}
    />
  );
}
```

---

### Common Form Inputs

```jsx
// Text input
<input value={text} onChange={(e) => setText(e.target.value)} />

// Textarea
<textarea value={text} onChange={(e) => setText(e.target.value)} />

// Select
<select value={option} onChange={(e) => setOption(e.target.value)}>
  <option value="a">A</option>
</select>

// Checkbox
<input
  type="checkbox"
  checked={isChecked}
  onChange={(e) => setIsChecked(e.target.checked)}
/>
```

---

### Multiple Form Fields

```jsx
function Form() {
  const [form, setForm] = useState({ name: "", email: "" });

  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));
  };

  return (
    <form>
      <input name="name" value={form.name} onChange={handleChange} />
      <input name="email" value={form.email} onChange={handleChange} />
    </form>
  );
}
```

---

## 11. Conditional Rendering

Render different content based on conditions:

---

### Ternary Operator

```jsx
<h1>{isLoggedIn ? "Welcome back!" : "Please log in"}</h1>
```

---

### Logical AND (&&)

```jsx
{isAdmin && <button>Delete User</button>}
```

---

### if/else Statements

```jsx
if (isLoggedIn) {
  return <h1>Welcome</h1>;
}

return <h1>Login</h1>;
```

---

## 12. Lists & Keys

### Rendering Lists

```jsx
const items = ["a", "b", "c"];

return (
  <ul>
    {items.map((item) => (
      <li key={item}>{item}</li>
    ))}
  </ul>
);
```

---

### Keys in Lists

Keys help React identify which items have changed:

```jsx
{users.map(user => (
  <li key={user.id}>{user.name}</li>
))}
```

---

### Key Best Practices

- Use stable, unique IDs: `key={item.id}`
- Avoid using array index as key in dynamic lists
- Avoid generating keys randomly

---

## 13. Component Lifecycle

With functional components, lifecycle is managed through hooks.

---

### Mount-Update-Unmount

- **Mount:** Component is created and inserted into DOM
- **Update:** Component re-renders due to state/prop changes
- **Unmount:** Component is removed from DOM

---

### useEffect Hook

```jsx
// Runs after every render
useEffect(() => {
  console.log("rendered");
});

// Runs only on mount
useEffect(() => {
  console.log("mounted");
}, []);

// Runs when dependencies change
useEffect(() => {
  console.log("count changed");
}, [count]);

// Cleanup on unmount
useEffect(() => {
  return () => {
    console.log("unmount");
  };
}, []);
```

---

## 14. Best Practices

**Keep components small:** One responsibility per component.

**Lift state up:** Share state in the nearest common parent.

**Use meaningful names:** Component names should describe their purpose.

**Props are immutable:** Never modify props.

**Extract logic:** Use custom hooks for reusable logic.

**Use developer tools:** React DevTools browser extension.

**Avoid premature optimization:** Profile before optimizing.
