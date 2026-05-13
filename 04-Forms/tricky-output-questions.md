## React Forms — Tricky Output Questions

> These questions test your understanding of controlled vs uncontrolled components, value/defaultValue semantics, onChange behavior, form submission, validation timing, and type coercion edge cases. Each scenario mirrors a real bug or interview question.

---

## 1. Controlled Components

### Q1

```jsx
function App() {
  return <input value="hello" />;
}
```

#### ❓ What happens when you try to type in this input?

<details>
<summary>✅ Answer</summary>

```txt
The input is frozen — you cannot type into it.
React logs a warning:
"Warning: You provided a `value` prop to a form field without an `onChange` handler."
```

**Explanation:** When you provide a `value` prop without an `onChange` handler, React locks the input. On every render, React sets the value back to `"hello"`. Any character you type is immediately overwritten. This is a read-only controlled input. To fix it: either add `onChange` (making it fully controlled) or add `readOnly` (making it intentionally read-only without the warning).

</details>

---

### Q2

```jsx
function App() {
  const [text, setText] = useState("hello");

  return (
    <div>
      <input value={text} onChange={e => setText(e.target.value)} />
      <input defaultValue={text} />
    </div>
  );
}
```

#### ❓ If `text` state later changes to `"world"`, what does each input show?

<details>
<summary>✅ Answer</summary>

```txt
First input: "world"   (controlled — always reflects state)
Second input: "hello"  (uncontrolled — ignores state changes after initial render)
```

**Explanation:** The first input is controlled: `value={text}` means React always sets the input value to the current state. When state changes to `"world"`, the input shows `"world"`. The second input uses `defaultValue`, which only sets the initial DOM value. After the first render, the DOM manages it independently. Subsequent state changes to `text` do not affect the uncontrolled input.

</details>

---

### Q3

```jsx
function App() {
  const [name, setName] = useState(undefined);

  return <input value={name} onChange={e => setName(e.target.value)} />;
}
```

#### ❓ What warning does React log? What happens when you start typing?

<details>
<summary>✅ Answer</summary>

```txt
React warning:
"Warning: A component is changing an uncontrolled input to be controlled."

After typing, the input works normally.
```

**Explanation:** When `value` is `undefined`, React treats the input as uncontrolled (same as having no `value` prop). When you type the first character, `onChange` fires, `setName` is called with a string value, and the input becomes controlled. This switches the input from uncontrolled to controlled mid-lifecycle, which React warns about. Fix: initialize state as an empty string: `useState("")`.

</details>

---

### Q4

```jsx
function App() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <input
        type="number"
        value={count}
        onChange={e => setCount(e.target.value)}
      />
      <p>Type: {typeof count}</p>
    </div>
  );
}
```

#### ❓ The initial type of `count` is `number`. After typing `5` in the input, what does `<p>` show?

<details>
<summary>✅ Answer</summary>

```txt
Type: string
```

**Explanation:** `e.target.value` is always a string, even for `type="number"`. When the user types `5`, `onChange` fires with `e.target.value = "5"` (a string). The state is set to `"5"`. The `<p>` shows `Type: string`. To keep the value as a number: `setCount(Number(e.target.value))` or `setCount(+e.target.value)`.

</details>

---

### Q5

```jsx
function App() {
  const [value, setValue] = useState("A");

  return (
    <input
      value={value}
      onChange={e => setValue(e.target.value.toUpperCase())}
    />
  );
}
```

#### ❓ The user types a lowercase `b`. What does the input display?

<details>
<summary>✅ Answer</summary>

```txt
"AB"
```

**Explanation:** The user types `b`. The `onChange` event fires with `e.target.value = "Ab"` (React merges the new character with the existing value). The handler calls `toUpperCase()`, producing `"AB"`. State is set to `"AB"`. React re-renders the input with `value="AB"`. The input shows `"AB"`. This is a live transformation pattern — every keystroke is intercepted and transformed before it appears.

</details>

---

## 2. Uncontrolled Components

### Q6

```jsx
function App() {
  const inputRef = useRef(null);

  console.log("During render:", inputRef.current);

  useEffect(() => {
    console.log("After mount:", inputRef.current);
  }, []);

  return <input ref={inputRef} defaultValue="hello" />;
}
```

#### ❓ What do the two console logs output?

<details>
<summary>✅ Answer</summary>

```txt
During render: null
After mount: <input> (the DOM element)
```

**Explanation:** During the render phase, the ref has not been assigned yet — the DOM element does not exist until after React commits. `inputRef.current` is `null` during render. After the component mounts, React assigns the DOM element to `inputRef.current`. The `useEffect` (which runs after mount) sees the actual DOM node. This is why you must never read `ref.current` during the render body for DOM elements — always use `useEffect` or event handlers.

</details>

---

### Q7

```jsx
function App() {
  const inputRef = useRef(null);

  const handleClick = () => {
    inputRef.current.value = "";    // clear via DOM
  };

  return (
    <div>
      <input ref={inputRef} defaultValue="initial" />
      <button onClick={handleClick}>Clear</button>
    </div>
  );
}
```

#### ❓ What happens when Clear is clicked? Is this pattern recommended?

<details>
<summary>✅ Answer</summary>

```txt
The input is cleared — it shows an empty string.
React is not aware of this change.

Not recommended: directly mutating the DOM bypasses React's state system.
```

**Explanation:** For uncontrolled inputs, you can manipulate the DOM directly via refs. Setting `inputRef.current.value = ""` clears the native DOM input. This works visually but React has no knowledge of the change. If you need to reset the form programmatically and track the value in React state, use a controlled component instead. For uncontrolled resets, this direct DOM mutation is the accepted approach.

</details>

---

### Q8

```jsx
function App() {
  const [key, setKey] = useState(0);

  return (
    <div>
      <input key={key} defaultValue="hello" />
      <button onClick={() => setKey(k => k + 1)}>Reset</button>
    </div>
  );
}
```

#### ❓ What happens when Reset is clicked?

<details>
<summary>✅ Answer</summary>

```txt
The input is unmounted and remounted with defaultValue "hello".
Any text the user typed is lost. The input resets to "hello".
```

**Explanation:** Changing the `key` prop causes React to unmount the old element and mount a new one. When the new `<input>` mounts, `defaultValue="hello"` is applied as the initial value. This is a common pattern for resetting uncontrolled inputs — force a remount by changing the key. It's simpler than tracking state and also resets all child inputs in a form group.

</details>

---

### Q9

```jsx
function App() {
  const fileRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    const file = fileRef.current.files[0];
    console.log(file?.name);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="file" ref={fileRef} />
      <button type="submit">Upload</button>
    </form>
  );
}
```

#### ❓ Can you add `value={someState}` to the file input to make it controlled? What does `files[0]` return if no file is selected?

<details>
<summary>✅ Answer</summary>

```txt
No — you cannot set value on a file input. Browsers block this for security.
React will warn if you try.

files[0] returns undefined if no file is selected.
```

**Explanation:** File inputs are always uncontrolled. The browser's security model prevents JavaScript from programmatically setting a file input's value (you cannot pre-fill a file). You can only read files the user has selected via `ref.current.files`. `files` is a `FileList` object. If no file is selected, `files` has length 0 and `files[0]` is `undefined`. Always guard with optional chaining: `file?.name`.

</details>

---

### Q10

```jsx
function App() {
  const ref = useRef(null);

  useEffect(() => {
    ref.current.focus();
  }, []);

  return (
    <div>
      <input ref={ref} />
      <button ref={ref}>Click me</button>
    </div>
  );
}
```

#### ❓ Which element gets focused? What does `ref.current` point to?

<details>
<summary>✅ Answer</summary>

```txt
The button gets focused.
ref.current points to the <button> element.
```

**Explanation:** A ref can only hold one reference at a time. When React renders, it processes refs in order. The `<input>` gets the ref first, so `ref.current` becomes the input element. Then the `<button>` gets the same ref, overwriting `ref.current` with the button. When `useEffect` runs, `ref.current` is the button element, so `focus()` focuses the button. Each element needs its own `useRef` instance to be referenced independently.

</details>

---

## 3. Form Submission

### Q11

```jsx
function App() {
  const handleSubmit = (e) => {
    console.log("Submitted");
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" type="email" defaultValue="" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ What happens when Submit is clicked?

<details>
<summary>✅ Answer</summary>

```txt
"Submitted" is logged AND the page reloads (form submits to server).
```

**Explanation:** `handleSubmit` does not call `e.preventDefault()`. The handler runs (logs "Submitted"), but then the browser's default form submission behavior fires — the page reloads and the form data is sent as a GET request to the current URL (since no `action` or `method` is specified). Always call `e.preventDefault()` in React form handlers to prevent this.

</details>

---

### Q12

```jsx
function App() {
  const handleSubmit = (e) => {
    e.preventDefault();
    const data = new FormData(e.target);
    console.log(data.get("username"));
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ The user types "Alice" and submits. What is logged?

<details>
<summary>✅ Answer</summary>

```txt
null
```

**Explanation:** `FormData` collects fields by their `name` attribute. The `<input>` has no `name` prop, so `FormData` does not include it. `data.get("username")` returns `null` because there is no field named "username". Fix: add `name="username"` to the input.

</details>

---

### Q13

```jsx
function App() {
  const [status, setStatus] = useState("idle");

  const handleSubmit = async (e) => {
    e.preventDefault();
    setStatus("loading");
    await new Promise(resolve => setTimeout(resolve, 1000));
    setStatus("done");
  };

  console.log("render:", status);

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={status === "loading"}>Submit</button>
    </form>
  );
}
```

#### ❓ List all console outputs in order after clicking Submit.

<details>
<summary>✅ Answer</summary>

```txt
render: idle        (initial render)
render: loading     (after setStatus("loading"))
render: done        (after setStatus("done"))
```

**Explanation:** On mount, the component renders with `status = "idle"`. When Submit is clicked, `handleSubmit` runs: `setStatus("loading")` triggers a re-render (`render: loading`). The button is now disabled. After the 1-second delay, `setStatus("done")` triggers another re-render (`render: done`). Each `setStatus` call schedules exactly one re-render.

</details>

---

### Q14

```jsx
function App() {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    await submitData();
    setIsSubmitting(false);
  };

  return (
    <form onSubmit={handleSubmit}>
      <button type="submit" disabled={isSubmitting}>Submit</button>
    </form>
  );
}
```

#### ❓ What happens if `submitData()` throws an error?

<details>
<summary>✅ Answer</summary>

```txt
setIsSubmitting(false) is NEVER called.
The button remains disabled forever (in the current session).
The error propagates as an unhandled promise rejection.
```

**Explanation:** If `await submitData()` throws, execution jumps out of the try block (or in this case, just the async function). `setIsSubmitting(false)` is never reached. The button stays disabled. Fix: use try/catch/finally — put `setIsSubmitting(false)` in the `finally` block so it always runs regardless of success or failure.

</details>

---

### Q15

```jsx
function App() {
  const [form, setForm] = useState({ email: "", password: "" });

  const handleChange = (e) => {
    setForm({ ...form, [e.target.name]: e.target.value });
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    // Simulate rapid state updates before submit
    setForm(prev => ({ ...prev, email: prev.email.trim() }));
    console.log(form.email);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" value={form.email} onChange={handleChange} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ The user types `"  alice@test.com  "` (with spaces). What does `console.log` print?

<details>
<summary>✅ Answer</summary>

```txt
"  alice@test.com  "  (with the surrounding spaces — NOT trimmed)
```

**Explanation:** `setForm(...)` schedules a state update but does not immediately change `form`. The `console.log(form.email)` reads the current render's snapshot of `form`, which still has the untrimmed value. The trim update will be reflected on the next render. This is the classic "stale closure" / "state as a snapshot" behavior. To log the trimmed value, compute it first: `const trimmed = form.email.trim(); setForm(...); console.log(trimmed);`

</details>

---

## 4. Validation

### Q16

```jsx
function App() {
  const [email, setEmail] = useState("");
  const [error, setError] = useState("");

  const validate = (value) => {
    if (!value) return "Required";
    if (!value.includes("@")) return "Invalid email";
    return "";
  };

  const handleChange = (e) => {
    const val = e.target.value;
    setEmail(val);
    setError(validate(val));
  };

  console.log("error:", error);

  return (
    <input value={email} onChange={handleChange} />
  );
}
```

#### ❓ The user types `a` then `@` then `b`. List all console outputs in order.

<details>
<summary>✅ Answer</summary>

```txt
error:          (initial render, empty string)
error: Invalid email    (after typing "a")
error:                  (after typing "@" → "a@" passes includes check)
error:                  (after typing "b" → "a@b" still passes)
```

**Explanation:** On initial render, `error` is `""`. After typing `"a"`, validation returns `"Invalid email"` (no `@`). After typing `"@"`, the value is `"a@"` — this passes `value.includes("@")`, so validation returns `""`. After typing `"b"`, value is `"a@b"` — still passes. Note that real email validation requires a more thorough regex. The key insight is that validation runs on every keystroke because `handleChange` calls `validate` and `setError` synchronously.

</details>

---

### Q17

```jsx
function App() {
  const [name, setName] = useState("");
  const [error, setError] = useState("");
  const [touched, setTouched] = useState(false);

  const handleSubmit = (e) => {
    e.preventDefault();
    if (!name) {
      setError("Name is required");
    } else {
      setError("");
      submitForm({ name });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}
        onChange={e => setName(e.target.value)}
      />
      {error && <span>{error}</span>}
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ The user submits without filling the field. The error appears. The user then types `"Alice"` and submits again. What happens to the error?

<details>
<summary>✅ Answer</summary>

```txt
On first submit (empty): error = "Name is required" — span shows error.
On second submit (with "Alice"): error = "" — span disappears.
```

**Explanation:** On the first submit, `name` is `""`, so `setError("Name is required")` runs. The span renders. When the user types, the name updates but the error state is not cleared on typing (no `onChange` clearing logic). However, on the second submit with `name = "Alice"`, the `else` branch runs: `setError("")` clears the error. The span condition `{error && ...}` is `false`, so it unmounts.

</details>

---

### Q18

```jsx
function App() {
  const [value, setValue] = useState("");
  const [isValid, setIsValid] = useState(false);

  const handleChange = (e) => {
    const v = e.target.value;
    setValue(v);
    setIsValid(v.length >= 3);
  };

  return (
    <div>
      <input value={value} onChange={handleChange} />
      <button type="button" disabled={!isValid} onClick={() => console.log("clicked")}>
        Submit
      </button>
    </div>
  );
}
```

#### ❓ The user types `"ab"` then `"c"` (making it `"abc"`). How many renders occur total? When does the button become enabled?

<details>
<summary>✅ Answer</summary>

```txt
4 renders total: initial + one per character typed (3 renders).
Button becomes enabled after typing the third character ("abc").
```

**Explanation:** Initial render (1). Typing `"a"`: `setValue("a")`, `setIsValid(false)` — React batches these in React 18, one re-render (2). Typing `"b"`: state becomes `"ab"`, `false` — one re-render (3). Typing `"c"`: state becomes `"abc"`, `true` — one re-render (4). The button's `disabled={!isValid}` becomes `disabled={false}` at render 4, enabling it. React 18 batches multiple `setState` calls in the same event handler into one render.

</details>

---

### Q19

```jsx
function App() {
  const [form, setForm] = useState({ email: "", agree: false });
  const [canSubmit, setCanSubmit] = useState(false);

  useEffect(() => {
    setCanSubmit(form.email.includes("@") && form.agree);
  }, [form]);

  return (
    <form>
      <input
        value={form.email}
        onChange={e => setForm(p => ({ ...p, email: e.target.value }))}
      />
      <input
        type="checkbox"
        checked={form.agree}
        onChange={e => setForm(p => ({ ...p, agree: e.target.checked }))}
      />
      <button disabled={!canSubmit}>Submit</button>
    </form>
  );
}
```

#### ❓ The user types `"a@b"` and checks the checkbox. How many renders occur before the button enables?

<details>
<summary>✅ Answer</summary>

```txt
At minimum 5 renders:
1. Initial render
2. After typing "a"
3. After effect runs → setCanSubmit(false) [no visible change since canSubmit was false]
4. After typing "@"
5. After effect runs → setCanSubmit(false) [still no @-only insufficient]
...continues for each character...
Then: checkbox checked → render → effect → setCanSubmit(true) → render
```

**Explanation:** Using `useEffect` to compute derived state causes extra renders. Every form change triggers: (1) a re-render from `setForm`, then (2) the effect runs and calls `setCanSubmit`, causing another re-render. This is a doubled render count. `canSubmit` is actually derived state — it should be computed directly: `const canSubmit = form.email.includes("@") && form.agree;`. This avoids the extra renders entirely.

</details>

---

### Q20

```jsx
function App() {
  const [password, setPassword] = useState("");
  const [confirm, setConfirm] = useState("");

  const errors = {
    password: password.length > 0 && password.length < 8
      ? "Min 8 characters"
      : "",
    confirm: confirm.length > 0 && confirm !== password
      ? "Passwords do not match"
      : "",
  };

  const isValid = !errors.password && !errors.confirm
    && password.length >= 8 && confirm.length > 0;

  return (
    <form>
      <input value={password} onChange={e => setPassword(e.target.value)} />
      {errors.password && <span>{errors.password}</span>}
      <input value={confirm} onChange={e => setConfirm(e.target.value)} />
      {errors.confirm && <span>{errors.confirm}</span>}
      <button disabled={!isValid}>Submit</button>
    </form>
  );
}
```

#### ❓ The user types `"password123"` in the first field and `"password12"` in the confirm field. What shows on screen?

<details>
<summary>✅ Answer</summary>

```txt
No error under the first field (length >= 8, no error).
"Passwords do not match" error under the confirm field.
Submit button is disabled.
```

**Explanation:** `password = "password123"` (11 chars) — `errors.password` is `""` (length check passes). `confirm = "password12"` (10 chars) — `errors.confirm` checks `confirm !== password` → `"password12" !== "password123"` is `true`, so error is `"Passwords do not match"`. `isValid` is `false` because `errors.confirm` is truthy. The button is disabled. This inline derived state pattern avoids `useState` for errors.

</details>

---

## 5. Edge Cases

### Q21

```jsx
function App() {
  const [quantity, setQuantity] = useState(1);

  return (
    <input
      type="number"
      value={quantity}
      onChange={e => setQuantity(e.target.value)}
      min={1}
      max={10}
    />
  );
}
```

#### ❓ The user clears the field (deletes `1`). What is the state value? What does the input show?

<details>
<summary>✅ Answer</summary>

```txt
State: ""  (empty string)
Input shows: empty (blank field)
```

**Explanation:** When the user clears the field, `e.target.value` is `""` (empty string). `setQuantity("")` stores an empty string in state. The `min` and `max` attributes are HTML validation attributes — they do not prevent the state update in a controlled React input. The `type="number"` attribute also does not prevent `e.target.value` from being `""`. To handle this correctly, use: `setQuantity(e.target.value === "" ? "" : Number(e.target.value))` and validate before submission.

</details>

---

### Q22

```jsx
function App() {
  const [checked, setChecked] = useState(false);

  return (
    <div>
      <input
        type="checkbox"
        value="newsletter"
        checked={checked}
        onChange={e => setChecked(e.target.value)}
      />
      <p>Checked state: {String(checked)}</p>
    </div>
  );
}
```

#### ❓ After clicking the checkbox once, what does the paragraph show?

<details>
<summary>✅ Answer</summary>

```txt
Checked state: newsletter
```

**Explanation:** The `onChange` handler uses `e.target.value` (which is `"newsletter"`, the `value` attribute of the checkbox) instead of `e.target.checked` (which would be the boolean `true`). So `setChecked("newsletter")` stores the string `"newsletter"`. The `{String(checked)}` renders `"newsletter"`. The visual appearance of the checkbox is now incorrect too — `checked="newsletter"` is truthy, so the checkbox appears checked, but the logic is wrong. Always use `e.target.checked` for checkboxes.

</details>

---

### Q23

```jsx
function App() {
  const [selected, setSelected] = useState("b");

  return (
    <select value={selected} onChange={e => setSelected(e.target.value)}>
      <option value="a">Option A</option>
      <option value="b">Option B</option>
      <option value="c">Option C</option>
    </select>
  );
}
```

#### ❓ Which option is initially selected in the dropdown?

<details>
<summary>✅ Answer</summary>

```txt
Option B
```

**Explanation:** In a controlled `<select>`, the `value` prop determines which option is selected. React matches the `value` prop (`"b"`) to the `<option>` with `value="b"` and marks it as selected. This replaces the HTML default behavior of using the `selected` attribute on an `<option>`. Never use `<option selected>` in React — use the `value` prop on the `<select>` element instead.

</details>

---

### Q24

```jsx
function App() {
  const [values, setValues] = useState([]);

  return (
    <select
      multiple
      value={values}
      onChange={e => {
        const options = e.target.options;
        const selected = [];
        for (let i = 0; i < options.length; i++) {
          if (options[i].selected) selected.push(options[i].value);
        }
        setValues(selected);
      }}
    >
      <option value="react">React</option>
      <option value="vue">Vue</option>
      <option value="angular">Angular</option>
    </select>
  );
}
```

#### ❓ Is this a correct pattern for a multi-select? What is an alternative shorter way to write the `onChange`?

<details>
<summary>✅ Answer</summary>

```txt
Yes, this is correct. The loop correctly collects all selected options.

Shorter alternative:
onChange={e =>
  setValues(Array.from(e.target.selectedOptions, opt => opt.value))
}
```

**Explanation:** For `<select multiple>`, `e.target.options` is an HTMLOptionsCollection and `e.target.selectedOptions` contains only the selected ones. The for-loop approach works but is verbose. `Array.from(e.target.selectedOptions, opt => opt.value)` is the idiomatic one-liner. Both approaches produce the same result: an array of the string values of currently selected options.

</details>

---

### Q25

```jsx
function App() {
  const [form, setForm] = useState({ name: "" });

  const handleSubmit = (e) => {
    e.preventDefault();
    const copy = form;
    copy.name = copy.name.toUpperCase();
    setForm(copy);
    console.log(form.name);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="name"
        value={form.name}
        onChange={e => setForm({ name: e.target.value })}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ The user types `"alice"` and submits. What does `console.log` print? Does the input update?

<details>
<summary>✅ Answer</summary>

```txt
console.log: "ALICE"
The input does NOT update to show "ALICE" on this render.
```

**Explanation:** `const copy = form` — `copy` and `form` are references to the same object. `copy.name = copy.name.toUpperCase()` mutates the original object. Now `form.name` is already `"ALICE"` before `setForm` is called. `console.log(form.name)` prints `"ALICE"` synchronously. However, because `copy` is the same reference as the previous state, React may bail out of re-rendering (same reference equality check). The input may not visually update. This is why you should never mutate state: `setForm({ name: form.name.toUpperCase() })` is the correct pattern.

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Controlled Components | Q1–Q5 | value without onChange, defaultValue vs value, undefined state warning, type coercion, live transformation |
| Uncontrolled Components | Q6–Q10 | ref before mount, defaultValue, DOM mutation, file input, overwritten ref |
| Form Submission | Q11–Q15 | preventDefault, FormData requires name, async render order, missing finally, stale snapshot |
| Validation | Q16–Q20 | real-time validation renders, error clear on submit, derived vs stored state, double render with useEffect, inline derived errors |
| Edge Cases | Q21–Q25 | number input empty string, checkbox value vs checked, select controlled, multi-select array, state mutation reference |
