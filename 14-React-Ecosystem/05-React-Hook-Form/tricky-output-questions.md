## React Hook Form — Tricky Output Questions

> These questions test deep understanding of React Hook Form's uncontrolled model, validation lifecycle, Controller behavior, formState subscriptions, and dynamic field arrays. Each question reflects real interview and debugging scenarios.

---

## 1. Basic Usage

### Q1

```tsx
import { useForm } from 'react-hook-form';

function App() {
  const { register } = useForm({ defaultValues: { name: '' } });

  console.log('render');

  return (
    <form>
      <input {...register('name')} />
    </form>
  );
}
```

User types 10 characters into the input. How many times is "render" logged?

#### ❓ How many re-renders occur when typing?

<details>
<summary>✅ Answer</summary>

```txt
render  ← initial mount only

Total: 1 render (no re-renders while typing)
```

**Explanation:** React Hook Form uses an uncontrolled approach. The `register` function provides a `ref` that gives RHF direct access to the DOM element's value — no React state is updated on each keystroke. The component does NOT re-render when the user types. Re-renders only happen for specific events: form submission, validation errors, `watch()` subscriptions, or explicit `setValue`/`reset` calls. This is the primary performance advantage of RHF over controlled forms.

</details>

---

### Q2

```tsx
function App() {
  const { register } = useForm({ defaultValues: { email: 'initial@example.com' } });

  return (
    <form>
      <input {...register('email')} id="email-input" />
    </form>
  );
}
```

What is the initial value shown in the input field when the component mounts?

#### ❓ What does the input display and how is the default value applied?

<details>
<summary>✅ Answer</summary>

```txt
The input shows: initial@example.com
```

**Explanation:** `defaultValues` in `useForm` sets the initial value of each field. For native HTML inputs registered with `register`, RHF sets the `defaultValue` prop on the input (React's `defaultValue` for uncontrolled inputs) or uses the ref to set the value after mount. The user sees `initial@example.com` in the field on first render. Unlike `value` (controlled), `defaultValue` doesn't cause re-renders when changed — it only sets the initial DOM value.

</details>

---

### Q3

```tsx
function App() {
  const { register, handleSubmit } = useForm<{ name: string }>();

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

User types "Alice" and submits. What is logged?

#### ❓ What does the `data` object contain when onSubmit is called?

<details>
<summary>✅ Answer</summary>

```txt
{ name: "Alice" }
```

**Explanation:** `handleSubmit` reads the current DOM value of each registered input via refs, collects them into an object keyed by field name, and passes the typed object to `onSubmit`. The field name `'name'` (from `register('name')`) becomes the key. The value is the current input value string `"Alice"`. `handleSubmit` also prevents the default form submission behavior automatically — no `e.preventDefault()` needed.

</details>

---

### Q4

```tsx
function App() {
  const { register, handleSubmit } = useForm<{
    age: number;
  }>();

  const onSubmit = (data) => console.log(typeof data.age, data.age);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input type="number" {...register('age')} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

User types `25` and submits. What is logged?

#### ❓ What is the type and value of `data.age`?

<details>
<summary>✅ Answer</summary>

```txt
"string" 25
```

**Explanation:** HTML `<input type="number">` returns its value as a string from the DOM. Without `{ valueAsNumber: true }`, RHF reads the string `"25"` from the input and passes it to `onSubmit`. The TypeScript type says `number` but the runtime value is a string. To actually get a number, use: `register('age', { valueAsNumber: true })`. This is a common gotcha when working with numeric inputs.

</details>

---

### Q5

```tsx
function App() {
  const { register, formState: { isDirty } } = useForm({
    defaultValues: { name: 'Alice' },
  });

  console.log('isDirty:', isDirty);

  return (
    <form>
      <input {...register('name')} />
    </form>
  );
}
```

User clears the input and types "Alice" back. Is the form dirty?

#### ❓ What is the value of `isDirty` after the user types "Alice" back?

<details>
<summary>✅ Answer</summary>

```txt
isDirty: false
```

**Explanation:** `isDirty` is `true` only when the CURRENT value differs from the `defaultValues`. The `defaultValues.name` is `'Alice'`. When the user clears and re-types `'Alice'`, the current value matches the default again. RHF compares current form values against `defaultValues` deeply using equality checks. `isDirty` reflects the presence of any actual change from the original defaults, not whether the user interacted with the form.

</details>

---

## 2. Validation

### Q6

```tsx
function App() {
  const { register, handleSubmit, formState: { errors } } = useForm<{
    email: string;
  }>({
    mode: 'onSubmit',  // default
  });

  return (
    <form onSubmit={handleSubmit((d) => console.log(d))}>
      <input {...register('email', { required: 'Email required' })} />
      {errors.email && <p id="err">{errors.email.message}</p>}
      <button type="submit">Submit</button>
    </form>
  );
}
```

User focuses the email field, types nothing, blurs it. Is `<p id="err">` visible?

#### ❓ Does the error message show after blur with `mode: 'onSubmit'`?

<details>
<summary>✅ Answer</summary>

```txt
No — the error does NOT show after blur.
The error only appears after the form is submitted.
```

**Explanation:** `mode: 'onSubmit'` (the default) means validation runs only when the form is submitted. Focusing and blurring a field does NOT trigger validation. The error appears only after the first submit attempt fails. If you want errors to show after the user leaves an empty required field, use `mode: 'onBlur'` or `mode: 'onTouched'`. After the first submission, `reValidateMode: 'onChange'` (default) will re-validate on each change.

</details>

---

### Q7

```tsx
function App() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    mode: 'onChange',
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input
        {...register('username', {
          validate: async (value) => {
            await new Promise(r => setTimeout(r, 1000));
            return value !== 'taken' || 'Username already taken';
          },
        })}
      />
      {errors.username && <p>{errors.username.message}</p>}
    </form>
  );
}
```

User types "taken". When does the error appear?

#### ❓ When is the validation error shown relative to when the user types?

<details>
<summary>✅ Answer</summary>

```txt
After 1 second delay — the async validate function runs on each change
(because mode: 'onChange'), and the error appears after the Promise resolves.
```

**Explanation:** With `mode: 'onChange'`, validation runs on every input change. The async `validate` function is called when the user types. The 1-second `setTimeout` simulates an API call (like checking username availability). During the 1-second wait, no error is shown. After 1000ms, the validate function returns `'Username already taken'`, RHF sets the error, and the component re-renders to show the error message. This pattern is used for server-side uniqueness checks.

</details>

---

### Q8

```tsx
function App() {
  const { register, handleSubmit, formState: { errors } } = useForm<{
    password: string;
  }>();

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input
        type="password"
        {...register('password', {
          validate: {
            hasUppercase: (v) => /[A-Z]/.test(v) || 'Need uppercase',
            minLength: (v) => v.length >= 8 || 'Min 8 chars',
          },
        })}
      />
      {errors.password && <p>{errors.password.message}</p>}
    </form>
  );
}
```

User submits with password `"hello"` (no uppercase, less than 8 chars). What error message shows?

#### ❓ Which validation error is shown? Both or just one?

<details>
<summary>✅ Answer</summary>

```txt
"Need uppercase"
Only the FIRST failing validation is shown (criteriaMode: 'firstError' default).
```

**Explanation:** With the default `criteriaMode: 'firstError'`, RHF stops at the first failing validate function and returns that error. Even though `minLength` also fails, only `hasUppercase`'s message `"Need uppercase"` is returned because it's listed first. To collect all errors simultaneously, use `criteriaMode: 'all'` in `useForm`. Then `errors.password.types` contains all failing rule types.

</details>

---

### Q9

```tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Invalid email'),
  age: z.number().min(18, 'Must be 18 or older'),
});

function App() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(schema),
    defaultValues: { email: '', age: 0 },
  });
}
```

User submits with `email: "not-an-email"` and `age: 15`. What validation rules fire?

#### ❓ Are BOTH validation errors reported simultaneously, or only the first?

<details>
<summary>✅ Answer</summary>

```txt
BOTH errors are reported simultaneously:
- errors.email.message = "Invalid email"
- errors.age.message = "Must be 18 or older"
```

**Explanation:** When using a schema resolver (Zod, Yup, Joi), the entire schema validates at once and all errors are collected and reported in a single pass. Unlike RHF's built-in sequential `validate` (which stops at first error per field with `criteriaMode: 'firstError'`), Zod validates all fields and returns all errors simultaneously. Each field's error appears independently in `errors.email` and `errors.age`.

</details>

---

### Q10

```tsx
function App() {
  const { register, trigger, formState: { errors } } = useForm<{
    email: string;
  }>({
    mode: 'onSubmit',
  });

  async function checkEmail() {
    const isValid = await trigger('email');
    console.log('email valid:', isValid);
  }

  return (
    <form>
      <input {...register('email', { required: 'Required' })} />
      {errors.email && <p>{errors.email.message}</p>}
      <button type="button" onClick={checkEmail}>Check Email</button>
    </form>
  );
}
```

The input is empty. User clicks "Check Email". What is logged and what does the user see?

#### ❓ What happens when `trigger('email')` is called manually?

<details>
<summary>✅ Answer</summary>

```txt
Logged: "email valid: false"
The error message "Required" appears in the UI.
```

**Explanation:** `trigger(fieldName)` manually runs validation for the specified field (or all fields if no argument) regardless of the current `mode`. It returns a Promise that resolves to `true` (valid) or `false` (invalid). After calling `trigger('email')` on an empty field with `required: 'Required'`, the error state is updated and the component re-renders to show the error. This is useful for validating a field before allowing the user to proceed to the next step in a multi-step form.

</details>

---

## 3. Controller

### Q11

```tsx
import { useForm, Controller } from 'react-hook-form';

function App() {
  const { control, handleSubmit } = useForm<{ rating: number }>({
    defaultValues: { rating: 0 },
  });

  return (
    <form onSubmit={handleSubmit((d) => console.log(d))}>
      <Controller
        name="rating"
        control={control}
        render={({ field }) => (
          <StarRating
            value={field.value}
            onChange={field.onChange}
          />
        )}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

User clicks 4 stars and submits. What is logged?

#### ❓ What is in the submitted data?

<details>
<summary>✅ Answer</summary>

```txt
{ rating: 4 }
```

**Explanation:** `Controller` manages a controlled component. `field.value` is the current value tracked by RHF (starts at `0` from `defaultValues`). When the user clicks 4 stars, `field.onChange(4)` is called, which updates RHF's internal state for the `rating` field to `4`. On submit, `handleSubmit` collects the RHF-tracked value (not a DOM ref) and passes `{ rating: 4 }` to the callback.

</details>

---

### Q12

```tsx
function App() {
  const { control } = useForm<{ name: string }>();

  return (
    <Controller
      name="name"
      control={control}
      render={({ field }) => {
        console.log('render', field.value);
        return <input {...field} />;
      }}
    />
  );
}
```

User types 3 characters. How many times does "render" log?

#### ❓ How many re-renders does Controller cause while typing?

<details>
<summary>✅ Answer</summary>

```txt
render (initial — empty string)
render (after 1st character)
render (after 2nd character)
render (after 3rd character)

Total: 4 renders
```

**Explanation:** `Controller` is a controlled component — it stores the value in React state (RHF's internal state). Every `field.onChange` call updates RHF's state and triggers a re-render of the Controller's `render` function. This is the key difference between `register` (uncontrolled, 0 re-renders while typing) and `Controller` (controlled, re-renders on every change). Use `Controller` only when necessary (for third-party components that don't accept a ref).

</details>

---

### Q13

```tsx
function App() {
  const { register, control } = useForm<{ name: string }>();

  return (
    <div>
      {/* Approach A */}
      <input {...register('name')} />

      {/* Approach B */}
      <Controller
        name="name"
        control={control}
        render={({ field }) => <input {...field} />}
      />
    </div>
  );
}
```

#### ❓ Which approach should be used for a plain native `<input>` and why?

<details>
<summary>✅ Answer</summary>

```txt
Approach A (register) is correct for plain native inputs.
Controller is unnecessary overhead for native inputs.
```

**Explanation:** `register` works directly with native HTML inputs via `ref`. It is uncontrolled — RHF reads the value through the DOM ref only when needed (validation, submit). `Controller` is designed for components that don't support `ref` forwarding (custom components, UI libraries). Using `Controller` with a plain `<input>` makes it controlled (re-renders on every change) without any benefit. Always prefer `register` for native inputs and `Controller`/`useController` for third-party controlled components.

</details>

---

### Q14

```tsx
function App() {
  const { control, handleSubmit } = useForm<{
    country: string;
  }>({
    defaultValues: { country: '' },
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="country"
        control={control}
        rules={{ required: 'Country is required' }}
        render={({ field, fieldState }) => (
          <div>
            <select {...field}>
              <option value="">Select country</option>
              <option value="US">United States</option>
            </select>
            {fieldState.error && <p>{fieldState.error.message}</p>}
          </div>
        )}
      />
      <button type="submit">Submit</button>
    </form>
  );
}
```

User submits without selecting a country. What is displayed?

#### ❓ How does `fieldState.error` differ from `formState.errors.country`?

<details>
<summary>✅ Answer</summary>

```txt
The error message "Country is required" is displayed.
fieldState.error and formState.errors.country contain the same error object.
```

**Explanation:** `fieldState.error` inside Controller's `render` prop is a convenience — it's the same as `formState.errors['country']`. `fieldState` provides `{ invalid, isTouched, isDirty, error }` scoped to the specific field being controlled. Using `fieldState.error` is cleaner inside Controller because you don't need to import `formState` or reference the field name. Both are equivalent: `fieldState.error === formState.errors.country`.

</details>

---

### Q15

```tsx
function App() {
  const { control, handleSubmit } = useForm<{
    tags: string[];
  }>({
    defaultValues: { tags: [] },
  });

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <Controller
        name="tags"
        control={control}
        render={({ field }) => (
          <TagInput
            value={field.value}    // string[]
            onChange={field.onChange}
          />
        )}
      />
    </form>
  );
}
```

`TagInput` calls `field.onChange(['react', 'typescript'])`. User submits. What is logged?

#### ❓ What shape does `field.onChange` accept?

<details>
<summary>✅ Answer</summary>

```txt
{ tags: ['react', 'typescript'] }
```

**Explanation:** `field.onChange` in Controller accepts the new value directly — not a synthetic event (unlike native input's `onChange` which provides an event). When you call `field.onChange(['react', 'typescript'])`, RHF stores the array as the value for the `tags` field. On submit, the typed object `{ tags: ['react', 'typescript'] }` is passed to `handleSubmit`'s callback. This is different from native inputs where `field.onChange(event)` extracts `event.target.value` automatically — Controller handles non-event values natively.

</details>

---

## 4. Form State

### Q16

```tsx
function App() {
  const { register, formState: { isDirty, dirtyFields }, setValue } = useForm({
    defaultValues: { name: 'Alice', age: 25 },
  });

  function handleClick() {
    setValue('name', 'Bob');
    console.log({ isDirty, dirtyFields });
  }

  return (
    <div>
      <input {...register('name')} />
      <input type="number" {...register('age')} />
      <button type="button" onClick={handleClick}>Set Name</button>
    </div>
  );
}
```

User clicks the button. What is logged?

#### ❓ What are `isDirty` and `dirtyFields` after calling `setValue('name', 'Bob')`?

<details>
<summary>✅ Answer</summary>

```txt
{ isDirty: false, dirtyFields: {} }
```

**Explanation:** By default, `setValue` does NOT mark the field as dirty. You must pass `{ shouldDirty: true }` to `setValue` for it to update `isDirty` and `dirtyFields`. Without that option: `setValue('name', 'Bob', { shouldDirty: true })` would give `{ isDirty: true, dirtyFields: { name: true } }`. Also note: the `console.log` reads the STALE closure values of `isDirty` and `dirtyFields` — they are from the render when the click handler was created. React state updates are asynchronous, so even with `shouldDirty: true`, the logged values would be from before the update.

</details>

---

### Q17

```tsx
function App() {
  const { register, formState: { isValid } } = useForm<{
    name: string;
    email: string;
  }>({
    mode: 'onChange',
    defaultValues: { name: '', email: '' },
  });

  console.log('isValid:', isValid);

  return (
    <form>
      <input {...register('name', { required: true })} />
      <input {...register('email', { required: true })} />
    </form>
  );
}
```

What is `isValid` on the initial render before the user has interacted with the form?

#### ❓ Is `isValid` true or false on first render with required fields and empty defaults?

<details>
<summary>✅ Answer</summary>

```txt
isValid: true  ← on initial render (before any interaction)
```

**Explanation:** RHF does NOT run validation on initial mount. Regardless of `mode`, form validation only runs after user interaction (typing, blur) or a submit attempt. On initial render, `isValid` defaults to `true` even when required fields are empty. This prevents showing "invalid" UI immediately when the form loads. After the user touches a field and it fails validation (in `'onChange'` mode), `isValid` becomes `false`. Use `isValid` to disable a Submit button only after the user has interacted with the form (`submitCount > 0 || isDirty`).

</details>

---

### Q18

```tsx
function App() {
  const {
    register,
    reset,
    formState: { isDirty, isSubmitted, submitCount },
  } = useForm({
    defaultValues: { email: '' },
  });

  function handleReset() {
    reset();
    console.log({ isDirty, isSubmitted, submitCount });
  }

  return (
    <div>
      <input {...register('email')} />
      <button type="button" onClick={handleReset}>Reset</button>
    </div>
  );
}
```

User types "test@example.com", submits (submitCount becomes 1), then clicks Reset. What is logged?

#### ❓ What are the formState values immediately after `reset()` is called?

<details>
<summary>✅ Answer</summary>

```txt
{ isDirty: true, isSubmitted: true, submitCount: 1 }
← These are STALE values from before the reset
```

After the re-render triggered by reset():
```txt
isDirty: false
isSubmitted: false
submitCount: 0
```

**Explanation:** The `console.log` inside `handleReset` reads the stale closure values from the current render cycle — before `reset()` triggers a re-render. `reset()` clears `isDirty`, `isSubmitted`, and `submitCount` to their initial states, but those changes are only visible in the NEXT render. The log captures the pre-reset values. This is the standard React stale closure behavior.

</details>

---

### Q19

```tsx
function App() {
  const { register, watch, formState: { touchedFields } } = useForm({
    defaultValues: { username: '', password: '' },
    mode: 'onBlur',
  });

  const username = watch('username');

  return (
    <form>
      <input {...register('username')} placeholder="Username" />
      <input type="password" {...register('password')} placeholder="Password" />
    </form>
  );
}
```

The form renders. User focuses and blurs the username field without typing. How many total renders have occurred?

#### ❓ How many renders happen due to `watch` and the blur event?

<details>
<summary>✅ Answer</summary>

```txt
3 renders total:
1. Initial mount
2. Watch subscription triggers on mount (internal initialization)
3. Blur event: touchedFields is updated → re-render
```

**Explanation:** `watch('username')` subscribes the component to username changes, which causes at least one additional render for the subscription setup. When the user blurs the field, `touchedFields.username` is set to `true`, which is part of `formState` and triggers a re-render. If the user had also typed something, each character would cause a re-render due to `watch`. This is why `watch` should be used sparingly — only subscribe to fields that genuinely need to drive UI changes.

</details>

---

### Q20

```tsx
function App() {
  const {
    register,
    handleSubmit,
    formState: { isSubmitting, isSubmitSuccessful },
  } = useForm<{ name: string }>();

  const onSubmit = async (data) => {
    await new Promise(r => setTimeout(r, 1000));
    throw new Error('Server error');
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name', { required: true })} defaultValue="Alice" />
      <button type="submit" disabled={isSubmitting}>Submit</button>
      <p>isSubmitSuccessful: {String(isSubmitSuccessful)}</p>
    </form>
  );
}
```

User submits the form. The `onSubmit` function throws after 1 second. What is `isSubmitSuccessful` after the error?

#### ❓ What is `isSubmitSuccessful` when `onSubmit` throws an error?

<details>
<summary>✅ Answer</summary>

```txt
isSubmitSuccessful: false
```

**Explanation:** `isSubmitSuccessful` is only `true` if `handleSubmit`'s callback resolves without throwing AND there are no validation errors. When `onSubmit` throws an error, RHF catches it (to prevent unhandled promise rejection) and sets `isSubmitSuccessful` to `false`. The error thrown inside `handleSubmit`'s callback is swallowed by RHF — you need to catch it yourself if you want to display a server error message (use a separate `useState` for that).

</details>

---

## 5. Edge Cases

### Q21

```tsx
function App() {
  const { register, control } = useForm<{
    items: { name: string }[];
  }>({
    defaultValues: {
      items: [{ name: 'A' }, { name: 'B' }, { name: 'C' }],
    },
  });

  const { fields, remove } = useFieldArray({ control, name: 'items' });

  return (
    <ul>
      {fields.map((field, index) => (
        <li key={field.id}>  {/* ← using field.id */}
          <input {...register(`items.${index}.name`)} />
          <button type="button" onClick={() => remove(index)}>Remove</button>
        </li>
      ))}
    </ul>
  );
}
```

There are 3 items: A, B, C. User removes item at index 1 (B). What do the remaining inputs display?

#### ❓ After removing item B (index 1), what are the remaining fields and their values?

<details>
<summary>✅ Answer</summary>

```txt
Remaining fields: [{ id: '...', name: 'A' }, { id: '...', name: 'C' }]
Input values: "A" and "C"
```

**Explanation:** `remove(1)` removes the item at index 1 (B) from the array. The array becomes `[{ name: 'A' }, { name: 'C' }]`. The two remaining inputs are re-rendered with indices 0 and 1. Since `field.id` (stable RHF-generated ID) is used as the key (not the index), React correctly updates only the removed item rather than causing all items to re-render with shifted values. If `index` were used as the key, item C would show in the B position's DOM node with B's previous value — a bug.

</details>

---

### Q22

```tsx
function App() {
  const { register, watch } = useForm({
    defaultValues: { type: 'A', details: '' },
  });

  const type = watch('type');
  console.log('render');

  return (
    <form>
      <select {...register('type')}>
        <option value="A">A</option>
        <option value="B">B</option>
      </select>
      {type === 'B' && <input {...register('details')} />}
    </form>
  );
}
```

User changes the select from "A" to "B". How many re-renders happen?

#### ❓ How many times is "render" logged after changing the select?

<details>
<summary>✅ Answer</summary>

```txt
render  ← initial mount
render  ← after selecting "B" (watch subscription triggers re-render)

Total: 2 renders
```

**Explanation:** `watch('type')` subscribes the component to changes in the `type` field. When the user changes the select to "B", the `type` watch subscription fires and causes one re-render. The component then renders with `type === 'B'`, showing the details input. Note that changing the select from "A" to "B" is one event and causes exactly one additional render — not multiple. This is the standard use case for `watch`: conditionally showing/hiding fields based on other field values.

</details>

---

### Q23

```tsx
function App() {
  const { register, getValues, handleSubmit } = useForm({
    defaultValues: { name: 'Alice', email: 'alice@test.com' },
  });

  function logValues() {
    const values = getValues();
    console.log(values);
  }

  return (
    <form onSubmit={handleSubmit(console.log)}>
      <input {...register('name')} />
      <input {...register('email')} />
      <button type="button" onClick={logValues}>Log Values</button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

User changes the name to "Bob" (without submitting), then clicks "Log Values". What is logged?

#### ❓ What does `getValues()` return after the user changes the name input?

<details>
<summary>✅ Answer</summary>

```txt
{ name: "Bob", email: "alice@test.com" }
```

**Explanation:** `getValues()` reads the CURRENT value of all registered inputs directly from the DOM refs. It does NOT require a React state update — it bypasses React and reads the DOM values directly. Even though the user changed the input and no re-render occurred (uncontrolled), `getValues()` still returns `"Bob"` because it reads from the actual DOM element's value property via the stored ref. This is what makes RHF both performant (no re-renders) and correct (can still read current values).

</details>

---

### Q24

```tsx
function App() {
  const { register, reset, formState: { defaultValues } } = useForm({
    defaultValues: { name: '' },
  });

  useEffect(() => {
    // Simulate data loaded from server after mount
    reset({ name: 'Server Name' });
  }, []);

  return (
    <form>
      <input {...register('name')} />
    </form>
  );
}
```

#### ❓ After `reset({ name: 'Server Name' })` runs, what is the new `defaultValues.name` and what is `isDirty` if the user types "Server Name" again?

<details>
<summary>✅ Answer</summary>

```txt
After reset: defaultValues.name = 'Server Name' (defaultValues updated by reset)
isDirty after typing 'Server Name': false
```

**Explanation:** `reset(newValues)` does two things: (1) sets the current field values to `newValues` AND (2) updates the internal `defaultValues` baseline. After `reset({ name: 'Server Name' })`, the new baseline for `isDirty` comparison is `{ name: 'Server Name' }`. If the user clears the field and retypes "Server Name", `isDirty` is `false` because the current value matches the updated baseline. This is the correct pattern for edit forms: use `reset(serverData)` so `isDirty` accurately reflects whether the user has made changes compared to the loaded server data.

</details>

---

### Q25

```tsx
function App() {
  const { register, handleSubmit } = useForm<{
    username: string;
  }>({
    shouldUnregister: true,
    defaultValues: { username: 'Alice' },
  });

  const [showField, setShowField] = useState(true);

  const onSubmit = (data) => console.log(data);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {showField && <input {...register('username')} />}
      <button type="button" onClick={() => setShowField(false)}>Hide</button>
      <button type="submit">Submit</button>
    </form>
  );
}
```

User clicks "Hide" (field unmounts), then submits. What is logged?

#### ❓ What does `data` contain when `shouldUnregister: true` and the field is unmounted?

<details>
<summary>✅ Answer</summary>

```txt
{} ← empty object, no 'username' key
```

**Explanation:** `shouldUnregister: true` tells RHF to remove a field from the form state when its input component unmounts. When the user clicks "Hide", the username input unmounts, and RHF unregisters the `username` field — removing its value and any validation rules from the form. On submit, the `username` key is absent from `data`. The default behavior (`shouldUnregister: false`) keeps the field registered even when unmounted, so `data` would contain `{ username: 'Alice' }`. Use `shouldUnregister: true` when unmounted fields should not contribute to the final form data.

</details>

---

## Topics Covered

| Category | Questions | Concepts Tested |
|---|---|---|
| Basic Usage | Q1–Q5 | Re-render count, defaultValues application, typed data, valueAsNumber, isDirty comparison |
| Validation | Q6–Q10 | mode: onSubmit vs onChange, async validation timing, criteriaMode firstError, Zod simultaneous errors, manual trigger |
| Controller | Q11–Q15 | Controller submit data, re-render count, when to use register vs Controller, fieldState vs formState errors, onChange with arrays |
| Form State | Q16–Q20 | setValue shouldDirty option, isValid initial state, reset clears formState, watch render count, isSubmitSuccessful on error |
| Edge Cases | Q21–Q25 | useFieldArray remove with field.id, watch conditional rendering, getValues reads DOM, reset updates defaultValues, shouldUnregister |
