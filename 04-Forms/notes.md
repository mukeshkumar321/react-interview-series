# React Forms

## Table of Contents

- [1. Controlled vs Uncontrolled Components](#1-controlled-vs-uncontrolled-components)
- [2. Controlled Input Patterns](#2-controlled-input-patterns)
- [3. Uncontrolled Components with useRef](#3-uncontrolled-components-with-useref)
- [4. Form Submission Handling](#4-form-submission-handling)
- [5. Multiple Inputs with a Single Handler](#5-multiple-inputs-with-a-single-handler)
- [6. Form Validation](#6-form-validation)
- [7. Error State Management](#7-error-state-management)
- [8. useReducer for Complex Forms](#8-usereducer-for-complex-forms)
- [9. File Inputs and File Handling](#9-file-inputs-and-file-handling)
- [10. Checkboxes and Radio Buttons](#10-checkboxes-and-radio-buttons)
- [11. Select Dropdown Patterns](#11-select-dropdown-patterns)
- [12. Textarea Handling](#12-textarea-handling)
- [13. Async Submission](#13-async-submission)
- [14. Controlled vs React Hook Form vs Formik](#14-controlled-vs-react-hook-form-vs-formik)
- [15. Common Mistakes](#15-common-mistakes)
- [16. Best Practices](#16-best-practices)

---

## 1. Controlled vs Uncontrolled Components

### Controlled Components

A controlled component is one where React state is the single source of truth for the input value. Every keystroke updates state, and the input reads its value from state.

```text
User types
    ↓
onChange fires
    ↓
setState called
    ↓
React re-renders
    ↓
Input value = state value
```

```jsx
function ControlledInput() {
  const [value, setValue] = useState("");

  return (
    <input
      value={value}                         // React state drives the display value
      onChange={e => setValue(e.target.value)}  // every change updates state
    />
  );
}
```

### Uncontrolled Components

An uncontrolled component lets the DOM manage the input value. React does not track the value on every keystroke. You read it only when needed (on submit) using a ref.

```text
User types
    ↓
DOM updates input value internally
    ↓
(React is not involved)
    ↓
On submit: ref.current.value reads from DOM
```

```jsx
function UncontrolledInput() {
  const inputRef = useRef(null);

  const handleSubmit = () => {
    console.log(inputRef.current.value);  // read on demand
  };

  return (
    <>
      <input ref={inputRef} defaultValue="initial" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

### Comparison Table

| Concern | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | DOM |
| Access value | `value` state variable | `ref.current.value` |
| Set initial value | `useState("")` | `defaultValue` prop |
| Real-time validation | Easy | Difficult |
| Dynamic disabling | Easy | Difficult |
| Instant value access | Yes | Only on read |
| Re-renders per keystroke | Yes | No |
| Integration with libraries | Better | Weaker |
| Performance on huge forms | Can be slow | Faster |

### When to Use Each

Use controlled when:
- You need real-time validation
- You need to conditionally disable fields
- You need to transform input as the user types (e.g., uppercase)
- You need to access the current value anywhere in the component
- You are using form state in other logic

Use uncontrolled when:
- Integrating with non-React code or libraries
- File inputs (always uncontrolled — you cannot set file input value via React)
- Simple forms with only a submit action and no live validation

---

## 2. Controlled Input Patterns

### Text Input

```jsx
function TextForm() {
  const [name, setName] = useState("");

  return (
    <input
      type="text"
      value={name}
      onChange={e => setName(e.target.value)}
      placeholder="Enter name"
    />
  );
}
```

### Number Input

```jsx
function NumberForm() {
  const [age, setAge] = useState(0);

  return (
    <input
      type="number"
      value={age}
      onChange={e => setAge(Number(e.target.value))}
    />
  );
}
```

Note: `e.target.value` is always a string, even for `type="number"`. Convert explicitly.

### Controlled Input with Transformation

```jsx
// Force uppercase as the user types
function UpperCaseInput() {
  const [code, setCode] = useState("");

  return (
    <input
      value={code}
      onChange={e => setCode(e.target.value.toUpperCase())}
    />
  );
}
```

### Controlled Input with Masking (Phone Number)

```jsx
function PhoneInput() {
  const [phone, setPhone] = useState("");

  const handleChange = (e) => {
    const digits = e.target.value.replace(/\D/g, "").slice(0, 10);
    const formatted = digits
      .replace(/(\d{3})(\d{3})(\d{4})/, "($1) $2-$3")
      .replace(/(\d{3})(\d{1,3})/, "($1) $2")
      .replace(/(\d{1,3})/, "($1");
    setPhone(formatted);
  };

  return <input value={phone} onChange={handleChange} />;
}
```

---

## 3. Uncontrolled Components with useRef

### Reading Input Value on Submit

```jsx
function LoginForm() {
  const usernameRef = useRef(null);
  const passwordRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    const username = usernameRef.current.value;
    const password = passwordRef.current.value;
    loginUser({ username, password });
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={usernameRef} type="text" defaultValue="" />
      <input ref={passwordRef} type="password" defaultValue="" />
      <button type="submit">Login</button>
    </form>
  );
}
```

### defaultValue vs value

| Prop | Behavior |
|---|---|
| `defaultValue` | Sets the initial value; DOM manages it after that (uncontrolled) |
| `value` | Sets the value on every render; React manages it (controlled) |

```jsx
// Uncontrolled — DOM takes over after initial render
<input defaultValue="initial text" />

// Controlled — React always drives the value
<input value={state} onChange={...} />

// Read-only (value without onChange) — React warns, input is frozen
<input value="frozen" />  // React warning: provide onChange or use readOnly
```

### Focusing with useRef

```jsx
function SearchBox() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus();  // focus on mount
  }, []);

  return <input ref={inputRef} type="search" />;
}
```

---

## 4. Form Submission Handling

### preventDefault

Without `preventDefault`, submitting an HTML form causes a full page reload (the browser's default behavior). Always call it in React forms.

```jsx
function ContactForm() {
  const [name, setName] = useState("");

  const handleSubmit = (e) => {
    e.preventDefault();   // stop browser default behavior
    console.log("Submitted:", name);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={e => setName(e.target.value)} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

### Collecting Form Data with FormData API

For uncontrolled forms with many fields, the `FormData` API is efficient:

```jsx
function ProfileForm() {
  const handleSubmit = (e) => {
    e.preventDefault();
    const formData = new FormData(e.target);

    const data = {
      name: formData.get("name"),
      email: formData.get("email"),
      bio: formData.get("bio"),
    };

    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" type="text" defaultValue="" />
      <input name="email" type="email" defaultValue="" />
      <textarea name="bio" defaultValue="" />
      <button type="submit">Save</button>
    </form>
  );
}
```

Note: Each input must have a `name` attribute for `FormData` to work.

### Button Types

| Type | Behavior |
|---|---|
| `type="submit"` | Submits the form |
| `type="reset"` | Resets form fields to default values |
| `type="button"` | No default behavior (use with onClick) |

---

## 5. Multiple Inputs with a Single Handler

### Using e.target.name

Instead of one `onChange` handler per field, use a single handler that reads `e.target.name` to know which field changed.

```jsx
function RegistrationForm() {
  const [form, setForm] = useState({
    firstName: "",
    lastName: "",
    email: "",
    age: "",
  });

  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));
  };

  return (
    <form>
      <input
        name="firstName"
        value={form.firstName}
        onChange={handleChange}
      />
      <input
        name="lastName"
        value={form.lastName}
        onChange={handleChange}
      />
      <input
        name="email"
        type="email"
        value={form.email}
        onChange={handleChange}
      />
      <input
        name="age"
        type="number"
        value={form.age}
        onChange={handleChange}
      />
    </form>
  );
}
```

### Handling Checkboxes in a Shared Handler

Checkboxes use `e.target.checked`, not `e.target.value`:

```jsx
const handleChange = (e) => {
  const { name, value, type, checked } = e.target;
  setForm(prev => ({
    ...prev,
    [name]: type === "checkbox" ? checked : value,
  }));
};
```

---

## 6. Form Validation

### HTML5 Built-in Validation

The simplest validation — let the browser handle it:

```jsx
<form>
  <input type="email" required minLength={3} maxLength={100} />
  <input type="number" min={18} max={120} />
  <input type="text" pattern="[A-Za-z]+" title="Letters only" />
  <button type="submit">Submit</button>
</form>
```

Limitation: styling is browser-dependent and customization is limited.

### Custom Validation on Submit

```jsx
function SignupForm() {
  const [form, setForm] = useState({ email: "", password: "" });
  const [errors, setErrors] = useState({});

  const validate = (values) => {
    const errs = {};
    if (!values.email) {
      errs.email = "Email is required";
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(values.email)) {
      errs.email = "Invalid email format";
    }
    if (!values.password) {
      errs.password = "Password is required";
    } else if (values.password.length < 8) {
      errs.password = "Password must be at least 8 characters";
    }
    return errs;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const errs = validate(form);
    if (Object.keys(errs).length > 0) {
      setErrors(errs);
      return;
    }
    // proceed with submission
    submitForm(form);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        value={form.email}
        onChange={e => setForm(p => ({ ...p, email: e.target.value }))}
      />
      {errors.email && <span className="error">{errors.email}</span>}

      <input
        name="password"
        type="password"
        value={form.password}
        onChange={e => setForm(p => ({ ...p, password: e.target.value }))}
      />
      {errors.password && <span className="error">{errors.password}</span>}

      <button type="submit">Sign Up</button>
    </form>
  );
}
```

### Real-Time Validation (onBlur)

Validate when the user leaves a field (loses focus), not on every keystroke:

```jsx
function EmailInput() {
  const [email, setEmail] = useState("");
  const [error, setError] = useState("");
  const [touched, setTouched] = useState(false);

  const validateEmail = (val) => {
    if (!val) return "Email is required";
    if (!/\S+@\S+\.\S+/.test(val)) return "Invalid email";
    return "";
  };

  return (
    <div>
      <input
        type="email"
        value={email}
        onChange={e => {
          setEmail(e.target.value);
          if (touched) setError(validateEmail(e.target.value));
        }}
        onBlur={() => {
          setTouched(true);
          setError(validateEmail(email));
        }}
      />
      {touched && error && <span>{error}</span>}
    </div>
  );
}
```

### Validation Strategy Comparison

| Strategy | When errors show | UX quality | Complexity |
|---|---|---|---|
| On submit only | After submit | Lowest | Lowest |
| On blur (field leave) | After user leaves field | Medium | Medium |
| On change (real-time) | While typing | Highest (can be annoying) | Medium |
| Hybrid (blur first, then change) | Best of both | Best | Higher |

---

## 7. Error State Management

### Per-Field Error State

```jsx
const [errors, setErrors] = useState({
  name: "",
  email: "",
  password: "",
});

// Set a single error
setErrors(prev => ({ ...prev, email: "Invalid email" }));

// Clear a single error
setErrors(prev => ({ ...prev, email: "" }));

// Clear all errors
setErrors({ name: "", email: "", password: "" });
```

### Error Display Patterns

```jsx
// Inline errors below each field
<div className="field">
  <label>Email</label>
  <input
    value={email}
    onChange={e => setEmail(e.target.value)}
    className={errors.email ? "input--error" : ""}
    aria-describedby="email-error"
    aria-invalid={!!errors.email}
  />
  {errors.email && (
    <span id="email-error" role="alert" className="error-message">
      {errors.email}
    </span>
  )}
</div>
```

### Accessibility in Error Messages

- Use `aria-invalid="true"` on the input when it has an error
- Use `aria-describedby` pointing to the error message element
- Use `role="alert"` on error messages so screen readers announce them
- Use `aria-live="polite"` for dynamic error regions

---

## 8. useReducer for Complex Forms

When forms have many fields, complex validation, and interdependent fields, `useReducer` provides more organized state management than many `useState` calls.

```jsx
const initialState = {
  values: { name: "", email: "", role: "user", bio: "" },
  errors: { name: "", email: "", bio: "" },
  touched: { name: false, email: false, bio: false },
  isSubmitting: false,
  isSuccess: false,
};

function formReducer(state, action) {
  switch (action.type) {
    case "FIELD_CHANGE":
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
      };
    case "FIELD_BLUR":
      return {
        ...state,
        touched: { ...state.touched, [action.field]: true },
        errors: { ...state.errors, [action.field]: action.error },
      };
    case "SET_ERRORS":
      return { ...state, errors: action.errors };
    case "SUBMIT_START":
      return { ...state, isSubmitting: true };
    case "SUBMIT_SUCCESS":
      return { ...state, isSubmitting: false, isSuccess: true };
    case "SUBMIT_FAIL":
      return { ...state, isSubmitting: false };
    case "RESET":
      return initialState;
    default:
      return state;
  }
}

function ComplexForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  const { values, errors, touched, isSubmitting } = state;

  const handleChange = (e) => {
    dispatch({ type: "FIELD_CHANGE", field: e.target.name, value: e.target.value });
  };

  const handleBlur = (e) => {
    const error = validateField(e.target.name, e.target.value);
    dispatch({ type: "FIELD_BLUR", field: e.target.name, error });
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    dispatch({ type: "SUBMIT_START" });
    try {
      await submitForm(values);
      dispatch({ type: "SUBMIT_SUCCESS" });
    } catch {
      dispatch({ type: "SUBMIT_FAIL" });
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" value={values.name} onChange={handleChange} onBlur={handleBlur} />
      {touched.name && errors.name && <span>{errors.name}</span>}

      <input name="email" value={values.email} onChange={handleChange} onBlur={handleBlur} />
      {touched.email && errors.email && <span>{errors.email}</span>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "Saving..." : "Save"}
      </button>
    </form>
  );
}
```

---

## 9. File Inputs and File Handling

File inputs are always uncontrolled — you cannot set `value` on a file input programmatically (browsers block it for security reasons).

### Reading a Single File

```jsx
function FileUpload() {
  const [file, setFile] = useState(null);

  const handleChange = (e) => {
    const selected = e.target.files[0];  // FileList is array-like
    setFile(selected || null);
  };

  return (
    <div>
      <input type="file" onChange={handleChange} accept="image/*" />
      {file && (
        <p>
          {file.name} — {(file.size / 1024).toFixed(1)} KB
        </p>
      )}
    </div>
  );
}
```

### File Preview

```jsx
function ImageUpload() {
  const [preview, setPreview] = useState(null);

  const handleChange = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const objectUrl = URL.createObjectURL(file);
    setPreview(objectUrl);

    // Revoke previous object URL to avoid memory leaks
    return () => URL.revokeObjectURL(objectUrl);
  };

  return (
    <div>
      <input type="file" accept="image/*" onChange={handleChange} />
      {preview && <img src={preview} alt="Preview" width={200} />}
    </div>
  );
}
```

### Uploading with FormData

```jsx
const handleSubmit = async (e) => {
  e.preventDefault();
  const file = e.target.avatar.files[0];

  const formData = new FormData();
  formData.append("avatar", file);
  formData.append("userId", "42");

  await fetch("/api/upload", {
    method: "POST",
    body: formData,
    // Do NOT set Content-Type header — browser sets it with boundary
  });
};
```

---

## 10. Checkboxes and Radio Buttons

### Single Checkbox

```jsx
function AgreementForm() {
  const [agreed, setAgreed] = useState(false);

  return (
    <label>
      <input
        type="checkbox"
        checked={agreed}
        onChange={e => setAgreed(e.target.checked)}  // use checked, not value
      />
      I agree to the terms
    </label>
  );
}
```

### Multiple Checkboxes (Array State)

```jsx
function TagSelector() {
  const [selectedTags, setSelectedTags] = useState([]);
  const allTags = ["React", "Vue", "Angular", "Svelte"];

  const handleToggle = (tag) => {
    setSelectedTags(prev =>
      prev.includes(tag)
        ? prev.filter(t => t !== tag)
        : [...prev, tag]
    );
  };

  return (
    <div>
      {allTags.map(tag => (
        <label key={tag}>
          <input
            type="checkbox"
            checked={selectedTags.includes(tag)}
            onChange={() => handleToggle(tag)}
          />
          {tag}
        </label>
      ))}
      <p>Selected: {selectedTags.join(", ")}</p>
    </div>
  );
}
```

### Radio Buttons

```jsx
function PlanSelector() {
  const [plan, setPlan] = useState("free");

  const plans = ["free", "pro", "enterprise"];

  return (
    <fieldset>
      <legend>Select Plan</legend>
      {plans.map(p => (
        <label key={p}>
          <input
            type="radio"
            name="plan"           // same name groups radios together
            value={p}
            checked={plan === p}
            onChange={e => setPlan(e.target.value)}
          />
          {p}
        </label>
      ))}
    </fieldset>
  );
}
```

---

## 11. Select Dropdown Patterns

### Single Select

```jsx
function LanguageSelector() {
  const [language, setLanguage] = useState("javascript");

  return (
    <select
      value={language}
      onChange={e => setLanguage(e.target.value)}
    >
      <option value="javascript">JavaScript</option>
      <option value="python">Python</option>
      <option value="rust">Rust</option>
    </select>
  );
}
```

### Select with Placeholder

```jsx
<select value={selected} onChange={e => setSelected(e.target.value)}>
  <option value="" disabled>Select a country</option>
  <option value="us">United States</option>
  <option value="uk">United Kingdom</option>
</select>
```

### Multi-Select

```jsx
function MultiSelect() {
  const [selected, setSelected] = useState([]);

  const handleChange = (e) => {
    // Convert HTMLOptionsCollection to array of selected values
    const values = Array.from(e.target.selectedOptions, opt => opt.value);
    setSelected(values);
  };

  return (
    <select
      multiple
      value={selected}
      onChange={handleChange}
      size={4}
    >
      <option value="react">React</option>
      <option value="vue">Vue</option>
      <option value="angular">Angular</option>
      <option value="svelte">Svelte</option>
    </select>
  );
}
```

---

## 12. Textarea Handling

In HTML, `<textarea>` content goes between the opening and closing tags. In React, use the `value` prop like any other input.

```jsx
// HTML approach (do NOT use in React)
<textarea>Initial text</textarea>

// React approach — use value and onChange
function TextareaForm() {
  const [bio, setBio] = useState("");
  const MAX = 500;

  return (
    <div>
      <textarea
        value={bio}
        onChange={e => setBio(e.target.value)}
        rows={5}
        maxLength={MAX}
      />
      <p>{bio.length}/{MAX} characters</p>
    </div>
  );
}
```

---

## 13. Async Submission

### Loading and Error States

```jsx
function ContactForm() {
  const [form, setForm] = useState({ name: "", message: "" });
  const [status, setStatus] = useState("idle"); // "idle" | "loading" | "success" | "error"
  const [errorMessage, setErrorMessage] = useState("");

  const handleSubmit = async (e) => {
    e.preventDefault();
    setStatus("loading");

    try {
      await submitContact(form);
      setStatus("success");
      setForm({ name: "", message: "" });  // reset on success
    } catch (err) {
      setStatus("error");
      setErrorMessage(err.message || "Something went wrong");
    }
  };

  if (status === "success") {
    return <p>Message sent! We will be in touch.</p>;
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={form.name}
        onChange={e => setForm(p => ({ ...p, name: e.target.value }))}
      />
      <textarea
        value={form.message}
        onChange={e => setForm(p => ({ ...p, message: e.target.value }))}
      />
      {status === "error" && <p className="error">{errorMessage}</p>}
      <button type="submit" disabled={status === "loading"}>
        {status === "loading" ? "Sending..." : "Send"}
      </button>
    </form>
  );
}
```

### Preventing Double Submission

```jsx
const [isSubmitting, setIsSubmitting] = useState(false);

const handleSubmit = async (e) => {
  e.preventDefault();
  if (isSubmitting) return;  // guard against double clicks

  setIsSubmitting(true);
  try {
    await submitForm(data);
  } finally {
    setIsSubmitting(false);  // always reset, even on error
  }
};

<button type="submit" disabled={isSubmitting}>
  {isSubmitting ? "Saving..." : "Save"}
</button>
```

---

## 14. Controlled vs React Hook Form vs Formik

### Feature Comparison

| Feature | Manual Controlled | React Hook Form | Formik |
|---|---|---|---|
| Bundle size | 0 (built-in) | ~9 KB | ~15 KB |
| Re-renders per keystroke | Yes | No (uncontrolled internally) | Yes |
| Validation library | Manual | Yup, Zod, custom | Yup, custom |
| Schema validation | Manual | Built-in with resolver | Built-in |
| Learning curve | None | Low | Medium |
| Performance | Slower on large forms | Fastest | Slower |
| TypeScript support | Manual | Excellent | Good |
| Setup effort | High | Low | Medium |

### React Hook Form Example

```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "At least 8 characters"),
});

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({ resolver: zodResolver(schema) });

  const onSubmit = async (data) => {
    await loginUser(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}

      <input type="password" {...register("password")} />
      {errors.password && <span>{errors.password.message}</span>}

      <button type="submit" disabled={isSubmitting}>Login</button>
    </form>
  );
}
```

### When to Choose

- Use manual controlled for simple forms with 1–3 fields
- Use React Hook Form for large forms, performance-critical forms, or TypeScript-heavy projects
- Use Formik when your team is already familiar with it or you need its wizard/multi-step form patterns

---

## 15. Common Mistakes

### Mistake 1: Missing preventDefault

```jsx
// ❌ Wrong — page reloads on submit
function Form() {
  const handleSubmit = () => {
    console.log("submitted");
  };
  return <form onSubmit={handleSubmit}>...</form>;
}

// ✅ Correct
const handleSubmit = (e) => {
  e.preventDefault();
  console.log("submitted");
};
```

### Mistake 2: Directly Mutating State Object

```jsx
// ❌ Wrong — mutates existing state object
const handleChange = (e) => {
  form.name = e.target.value;    // mutation, no re-render
  setForm(form);                 // same reference, React bails out
};

// ✅ Correct — create new object
const handleChange = (e) => {
  setForm(prev => ({ ...prev, name: e.target.value }));
};
```

### Mistake 3: Reading e.target.value Asynchronously

```jsx
// ❌ Wrong — e.target becomes null by the time async runs
const handleChange = (e) => {
  setTimeout(() => {
    setState(e.target.value);  // e.target might be null or re-used
  }, 0);
};

// ✅ Correct — extract value synchronously
const handleChange = (e) => {
  const value = e.target.value;  // capture before async operation
  setTimeout(() => {
    setState(value);
  }, 0);
};
```

### Mistake 4: Value Without onChange (Read-Only Input)

```jsx
// ❌ Wrong — React warns: "You provided a `value` prop without an `onChange` handler"
<input value={someValue} />

// ✅ Correct — either make it controlled
<input value={someValue} onChange={e => setValue(e.target.value)} />

// ✅ Or explicitly mark as read-only
<input value={someValue} readOnly />
```

### Mistake 5: Using value on File Input

```jsx
// ❌ Wrong — you cannot control file input value
<input type="file" value={fileState} onChange={...} />

// ✅ Correct — file input is always uncontrolled
<input type="file" onChange={e => setFile(e.target.files[0])} />
```

### Mistake 6: Forgetting to Reset Form After Submit

```jsx
// ❌ Wrong — form keeps old values after successful submit
const handleSubmit = async (e) => {
  e.preventDefault();
  await submit(form);
  // nothing clears the form
};

// ✅ Correct
const handleSubmit = async (e) => {
  e.preventDefault();
  await submit(form);
  setForm({ name: "", email: "" });  // reset after success
};
```

---

## 16. Best Practices

### 1. Use Semantic HTML

Wrap related inputs in `<fieldset>` with `<legend>`, use `<label>` with `htmlFor`, and use appropriate `type` attributes.

```jsx
<fieldset>
  <legend>Contact Information</legend>
  <label htmlFor="email">Email</label>
  <input id="email" type="email" name="email" />
</fieldset>
```

### 2. Associate Labels with Inputs

```jsx
// ✅ With htmlFor — clicking the label focuses the input
<label htmlFor="username">Username</label>
<input id="username" type="text" />

// ✅ Or wrap input inside label
<label>
  Username
  <input type="text" />
</label>
```

### 3. Show Errors Only After Interaction

Do not show errors on a field the user has not touched yet. Track a `touched` state per field.

### 4. Disable Submit During Loading

Always set `disabled={isSubmitting}` on the submit button to prevent double submission.

### 5. Handle the Finally Block

```jsx
// Always reset loading state regardless of success/failure
try {
  await submit();
} finally {
  setIsSubmitting(false);
}
```

### 6. Accessibility Checklist

- `aria-required="true"` on required fields
- `aria-invalid={!!error}` when validation fails
- `aria-describedby` pointing to error message
- `role="alert"` on error messages for live region announcements
- Focus the first error field after failed submit

### 7. Validate on the Server Too

Client-side validation is for UX only. Always re-validate on the server — never trust client input.
