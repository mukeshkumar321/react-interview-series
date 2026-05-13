# React Hook Form

## Table of Contents

1. [What is React Hook Form and Why Use It](#1-what-is-react-hook-form-and-why-use-it)
2. [Installation and Basic Setup](#2-installation-and-basic-setup)
3. [useForm Hook](#3-useform-hook)
4. [register Function](#4-register-function)
5. [handleSubmit](#5-handlesubmit)
6. [formState](#6-formstate)
7. [Controller Component](#7-controller-component)
8. [useController Hook](#8-usecontroller-hook)
9. [watch](#9-watch)
10. [setValue and getValues](#10-setvalue-and-getvalues)
11. [reset](#11-reset)
12. [Validation](#12-validation)
13. [useFieldArray](#13-usefieldarray)
14. [Form Submission States](#14-form-submission-states)
15. [Performance Comparison](#15-performance-comparison)
16. [Common Mistakes](#16-common-mistakes)
17. [Best Practices](#17-best-practices)

---

## 1. What is React Hook Form and Why Use It

React Hook Form (RHF) is a performant, flexible form library built on **uncontrolled inputs** with optional HTML5 validation. It minimizes re-renders to the absolute minimum by not using React state to track every field value change.

### The Problem with Controlled Forms

Traditional controlled forms re-render on every keystroke:

```jsx
// ❌ Controlled form — re-renders on EVERY keystroke
function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  // User types one character in email → re-render
  // User types one character in password → re-render
  // 50 keystrokes = 50 re-renders

  return (
    <form>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      <input value={password} onChange={e => setPassword(e.target.value)} />
    </form>
  );
}
```

For large forms (20+ fields), this creates significant performance problems.

### How RHF Works

RHF manages form state using **refs** instead of React state. The input values live in the DOM (the input elements themselves), not in React state. React state is only updated for specific operations (errors, validation state, `watch`).

```text
User types → DOM input value updates → NO React re-render
Submit → RHF reads values from refs → validates → calls onSubmit
Error occurs → RHF sets error state → components with errors re-render
```

### Why Use React Hook Form

| Concern | Without RHF | With RHF |
|---|---|---|
| Re-renders per keystroke | 1 per field | 0 (uncontrolled) |
| Code verbosity | High (useState + onChange per field) | Low (register one-liner) |
| Validation | Manual logic | Built-in rules + Yup/Zod integration |
| Error handling | Manual state | Automatic via `formState.errors` |
| Async validation | Complex | Simple async validate function |
| TypeScript | Manual typing | Inferred types from schema |
| Dynamic fields | Complex | `useFieldArray` built-in |

---

## 2. Installation and Basic Setup

```bash
npm install react-hook-form
```

### Minimal Example

```tsx
import { useForm } from 'react-hook-form';

type LoginFormValues = {
  email: string;
  password: string;
};

function LoginForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<LoginFormValues>();

  function onSubmit(data: LoginFormValues) {
    console.log(data);  // { email: '...', password: '...' }
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email', { required: 'Email is required' })} />
      {errors.email && <p>{errors.email.message}</p>}

      <input type="password" {...register('password', { required: 'Password is required' })} />
      {errors.password && <p>{errors.password.message}</p>}

      <button type="submit">Login</button>
    </form>
  );
}
```

---

## 3. useForm Hook

`useForm` is the main hook. It returns all the methods and state needed to build a form.

### Full Options

```ts
const form = useForm<FormValues>({
  // ─── Default Values ──────────────────────────────────────────────────────
  defaultValues: {
    email: '',
    age: 0,
    role: 'user',
  },

  // ─── Validation Mode ─────────────────────────────────────────────────────
  mode: 'onSubmit',        // 'onSubmit' | 'onBlur' | 'onChange' | 'onTouched' | 'all'
  reValidateMode: 'onChange', // when to re-validate after first submission

  // ─── Resolver (external validation) ──────────────────────────────────────
  resolver: zodResolver(schema),  // or yupResolver, joiResolver

  // ─── Misc ────────────────────────────────────────────────────────────────
  shouldFocusError: true,  // auto-focus first invalid field on submit
  criteriaMode: 'firstError',  // 'firstError' | 'all' (collect all errors per field)
  delayError: 500,         // delay showing error in ms (reduces flicker on fast typing)
});
```

### Destructuring Common Methods

```ts
const {
  register,        // Register a field with validation rules
  handleSubmit,    // Wrap your submit handler
  formState,       // Errors, isSubmitting, isDirty, isValid, etc.
  watch,           // Subscribe to field value changes
  setValue,        // Programmatically set a field value
  getValues,       // Read current field values without subscribing
  reset,           // Reset form to defaults or new values
  trigger,         // Manually trigger validation
  setError,        // Programmatically set a field error
  clearErrors,     // Clear specific or all errors
  setFocus,        // Focus a specific field
  control,         // Required for Controller component
} = useForm<FormValues>();
```

---

## 4. register Function

`register` connects an HTML input to RHF. It returns props to spread onto the input.

### What register Returns

```ts
const { onChange, onBlur, name, ref } = register('fieldName', rules);

// Spread onto input:
<input {...register('email')} />
// Equivalent to:
<input onChange={onChange} onBlur={onBlur} name="email" ref={ref} />
```

The `ref` is a callback ref that RHF uses to directly access the DOM element's value — no React state needed.

### Validation Rules

```tsx
// Required
<input {...register('name', {
  required: 'Name is required',  // or just true (no message)
})} />

// Min/Max for numbers
<input type="number" {...register('age', {
  min: { value: 18, message: 'Must be at least 18' },
  max: { value: 100, message: 'Must be under 100' },
})} />

// MinLength/MaxLength for strings
<input {...register('username', {
  minLength: { value: 3, message: 'At least 3 characters' },
  maxLength: { value: 20, message: 'At most 20 characters' },
})} />

// Pattern (regex)
<input {...register('email', {
  pattern: {
    value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
    message: 'Invalid email address',
  },
})} />

// Custom validate function
<input {...register('username', {
  validate: (value) => {
    if (value.includes(' ')) return 'No spaces allowed';
    return true;  // ← must return true for valid
  },
})} />

// Multiple validate rules
<input {...register('password', {
  validate: {
    hasUppercase: (v) => /[A-Z]/.test(v) || 'Need uppercase letter',
    hasNumber: (v) => /[0-9]/.test(v) || 'Need a number',
    minLength: (v) => v.length >= 8 || 'At least 8 characters',
  },
})} />

// Async validate
<input {...register('username', {
  validate: async (value) => {
    const exists = await checkUsernameExists(value);
    return !exists || 'Username already taken';
  },
})} />
```

### valueAsNumber and valueAsDate

```tsx
// Without valueAsNumber: input returns string "42"
<input type="number" {...register('age')} />

// With valueAsNumber: input returns number 42
<input type="number" {...register('age', { valueAsNumber: true })} />

// Date
<input type="date" {...register('birthday', { valueAsDate: true })} />
```

---

## 5. handleSubmit

`handleSubmit` is a higher-order function. It validates the form, then calls your callback with the typed data.

```tsx
const onSubmit = async (data: FormValues) => {
  // data is fully typed and validated
  await createUser(data);
};

const onError = (errors: FieldErrors<FormValues>) => {
  // Called when validation fails (optional second argument)
  console.log('Validation failed:', errors);
};

<form onSubmit={handleSubmit(onSubmit, onError)}>
```

### handleSubmit Behavior

```text
User clicks Submit
     ↓
handleSubmit intercepts the native submit event
     ↓
Runs all registered validators (including async)
     ↓
  ┌────────────────────────────────────────────┐
  │ Valid                   │ Invalid           │
  │ calls onSubmit(data)    │ sets errors       │
  │ isSubmitting: true      │ calls onError     │
  │                         │ focuses first err  │
  └────────────────────────────────────────────┘
     ↓
isSubmitting: false (after onSubmit resolves/rejects)
```

```tsx
// handleSubmit prevents default form submission automatically
// You do NOT need e.preventDefault()
<form onSubmit={handleSubmit(async (data) => {
  await saveToServer(data);
  // isSubmitting is true during this await
  // isSubmitSuccessful becomes true after this resolves
})}>
```

---

## 6. formState

`formState` contains metadata about the form's current state.

### All formState Properties

```ts
const {
  formState: {
    errors,             // Error objects per field: { email: { type, message } }
    isSubmitting,       // true while handleSubmit's async callback is running
    isSubmitted,        // true after first submit attempt
    isSubmitSuccessful, // true if last submit succeeded (no errors, callback resolved)
    isDirty,            // true if any field differs from defaultValues
    isValid,            // true if there are no errors
    isLoading,          // true if defaultValues is an async function loading
    touchedFields,      // { email: true } — fields that have been focused+blurred
    dirtyFields,        // { email: true } — fields that differ from defaultValues
    submitCount,        // number of times the form was submitted
  }
} = useForm();
```

### Using formState in UI

```tsx
function RegistrationForm() {
  const { register, handleSubmit, formState } = useForm<FormValues>({
    defaultValues: { name: '', email: '' },
  });
  const { errors, isSubmitting, isDirty, isValid } = formState;

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name', { required: true })} />
      {errors.name && <p>Name is required</p>}

      <input {...register('email')} />

      <button
        type="submit"
        disabled={isSubmitting || !isDirty || !isValid}
      >
        {isSubmitting ? 'Saving...' : 'Save'}
      </button>
    </form>
  );
}
```

### formState Subscriptions and Renders

RHF uses a subscription model. Only the fields of `formState` you actually destructure are subscribed. Unused fields don't cause re-renders:

```ts
// Only subscribes to errors and isSubmitting — no re-render for isDirty changes
const { errors, isSubmitting } = formState;

// Subscribes to all — re-renders on any formState change
const { errors, isSubmitting, isDirty, isValid, touchedFields } = formState;
```

---

## 7. Controller Component

`Controller` is used to integrate **controlled third-party components** or UI library components (MUI, Ant Design, React Select, custom inputs) that don't expose a native `ref`.

```tsx
import { useForm, Controller } from 'react-hook-form';
import Select from 'react-select';

function ProfileForm() {
  const { control, handleSubmit } = useForm<{
    role: { value: string; label: string };
    birthDate: Date | null;
  }>();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* React Select — doesn't work with register (no native ref) */}
      <Controller
        name="role"
        control={control}
        rules={{ required: 'Role is required' }}
        render={({ field, fieldState }) => (
          <>
            <Select
              {...field}  // value, onChange, onBlur, ref, name
              options={[
                { value: 'admin', label: 'Admin' },
                { value: 'user', label: 'User' },
              ]}
            />
            {fieldState.error && <p>{fieldState.error.message}</p>}
          </>
        )}
      />
    </form>
  );
}
```

### What Controller Provides

The `render` prop receives:
- `field` — `{ onChange, onBlur, value, name, ref }` — connect to the component
- `fieldState` — `{ invalid, isTouched, isDirty, error }` — field-specific state
- `formState` — full form state (same as useForm's formState)

### Controller vs register

| | `register` | `Controller` |
|---|---|---|
| Works with | Native HTML inputs | Controlled components / UI libraries |
| Value tracking | DOM ref (uncontrolled) | React state (controlled) |
| Re-renders | On error/validation only | On every value change |
| Ref access | Direct DOM ref | Forwarded ref or none needed |

---

## 8. useController Hook

`useController` provides the same functionality as `Controller` but as a hook — useful for building **reusable form components**:

```tsx
import { useController, Control } from 'react-hook-form';

interface TextFieldProps {
  name: string;
  control: Control<any>;
  label: string;
  rules?: object;
}

function TextField({ name, control, label, rules }: TextFieldProps) {
  const {
    field: { onChange, onBlur, value, ref },
    fieldState: { error },
  } = useController({ name, control, rules });

  return (
    <div>
      <label>{label}</label>
      <input
        onChange={onChange}
        onBlur={onBlur}
        value={value}
        ref={ref}
        className={error ? 'error' : ''}
      />
      {error && <p>{error.message}</p>}
    </div>
  );
}

// Usage
function MyForm() {
  const { control, handleSubmit } = useForm();
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <TextField name="email" control={control} label="Email" rules={{ required: true }} />
    </form>
  );
}
```

---

## 9. watch

`watch` subscribes to field value changes and causes re-renders when the watched field(s) change.

### Watching Individual Fields

```tsx
const { register, watch } = useForm<{ email: string; type: string }>();

const emailValue = watch('email');          // subscribe to email changes
const typeValue = watch('type');            // subscribe to type changes
const allValues = watch();                  // subscribe to all fields
const [email, type] = watch(['email', 'type']); // subscribe to multiple
```

### Conditional Rendering with watch

```tsx
function SignupForm() {
  const { register, watch } = useForm<{
    accountType: 'personal' | 'business';
    companyName: string;
  }>();

  const accountType = watch('accountType');

  return (
    <form>
      <select {...register('accountType')}>
        <option value="personal">Personal</option>
        <option value="business">Business</option>
      </select>

      {accountType === 'business' && (
        <input
          {...register('companyName', { required: 'Company name required' })}
          placeholder="Company name"
        />
      )}
    </form>
  );
}
```

### watch vs getValues

```ts
// watch — subscribes, causes re-render on change
const name = watch('name');  // component re-renders when 'name' changes

// getValues — reads value once, does NOT subscribe, NO re-render
const name = getValues('name');  // good for reading in event handlers
const all = getValues();          // good for reading inside handleSubmit
```

---

## 10. setValue and getValues

### setValue

Programmatically set a field value. Does NOT trigger re-render by default.

```tsx
const { register, setValue, getValues } = useForm<{
  name: string;
  tags: string[];
}>();

// Basic set
setValue('name', 'Alice');

// With validation trigger
setValue('email', 'alice@example.com', {
  shouldValidate: true,   // trigger validation after setting
  shouldDirty: true,      // mark field as dirty
  shouldTouch: true,      // mark field as touched
});

// Set array field
setValue('tags', ['react', 'typescript']);
```

### Common setValue Patterns

```tsx
// Pre-fill form from API data
useEffect(() => {
  if (userData) {
    setValue('name', userData.name);
    setValue('email', userData.email);
    // Or use reset() to set all fields at once
  }
}, [userData]);

// Clear a field
setValue('search', '');

// Set a field value based on another field
const country = watch('country');
useEffect(() => {
  if (country === 'US') setValue('currency', 'USD');
  if (country === 'EU') setValue('currency', 'EUR');
}, [country]);
```

### getValues

Reads current form values without re-rendering:

```tsx
// Read single field
const name = getValues('name');

// Read multiple fields
const { name, email } = getValues(['name', 'email'] as const);

// Read all fields (shallow copy)
const allValues = getValues();

// Good use: inside event handlers where you don't want to watch
function copyToClipboard() {
  const email = getValues('email');  // no re-render
  navigator.clipboard.writeText(email);
}
```

---

## 11. reset

`reset` clears the form back to default values (or new values you provide).

```tsx
const { register, reset, handleSubmit } = useForm<FormValues>({
  defaultValues: { name: '', email: '' },
});

// Reset to initial defaultValues
reset();

// Reset to specific values
reset({ name: 'Alice', email: 'alice@example.com' });

// Reset with options
reset(
  { name: '' },
  {
    keepErrors: false,     // clear errors (default)
    keepDirty: false,      // reset dirty state (default)
    keepIsSubmitted: false,
    keepTouched: false,
    keepIsValid: false,
    keepDefaultValues: false,  // keep original defaultValues after reset
  }
);
```

### Common reset Patterns

```tsx
// Reset after successful submit
const onSubmit = async (data: FormValues) => {
  await createUser(data);
  reset();  // clear form after success
};

// Reset to loaded server data (for edit forms)
useEffect(() => {
  if (userData) {
    reset(userData);  // fills form with server data AND updates defaultValues
    // Now isDirty correctly reflects changes against loaded server data
  }
}, [userData, reset]);
```

### reset vs setValue

```ts
// reset — clears ALL fields, resets ALL formState (errors, dirty, touched)
reset({ name: 'Alice' });

// setValue — sets ONE field, preserves other fields and formState
setValue('name', 'Alice');
```

---

## 12. Validation

### Built-in Validation Rules

```tsx
register('age', {
  required: 'Required',
  min: { value: 0, message: 'Must be positive' },
  max: { value: 120, message: 'Too high' },
  minLength: { value: 2, message: 'Too short' },
  maxLength: { value: 100, message: 'Too long' },
  pattern: { value: /^\d+$/, message: 'Digits only' },
  validate: (v) => v !== 'forbidden' || 'This value is not allowed',
})
```

### Custom Validation Function

```tsx
register('username', {
  validate: {
    noSpaces: (v) => !/\s/.test(v) || 'No spaces allowed',
    noSpecialChars: (v) => /^[a-zA-Z0-9_]+$/.test(v) || 'Only letters, numbers, underscore',
  },
})
```

### Async Validation

```tsx
register('email', {
  validate: async (email) => {
    if (!email) return true;  // skip if empty (let required handle it)
    const exists = await checkEmailExists(email);
    return !exists || 'Email already registered';
  },
})
```

### Yup Schema Validation

```bash
npm install @hookform/resolvers yup
```

```tsx
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';

const schema = yup.object({
  name: yup.string().required('Name is required').min(2),
  email: yup.string().email('Invalid email').required(),
  age: yup.number().min(18, 'Must be 18+').required(),
}).required();

function RegistrationForm() {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: yupResolver(schema),
  });
  // ...
}
```

### Zod Schema Validation

```bash
npm install @hookform/resolvers zod
```

```tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  name: z.string().min(2, 'At least 2 characters').max(50),
  email: z.string().email('Invalid email'),
  age: z.number().min(18, 'Must be 18+'),
  password: z.string()
    .min(8, 'At least 8 characters')
    .regex(/[A-Z]/, 'Need uppercase'),
  confirmPassword: z.string(),
}).refine(
  (data) => data.password === data.confirmPassword,
  { message: 'Passwords do not match', path: ['confirmPassword'] }
);

type FormValues = z.infer<typeof schema>;  // TypeScript type inferred from schema

function RegisterForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(schema),
  });
  // ...
}
```

### Validation Modes

```ts
// mode: 'onSubmit' (default) — validate only on form submit
// mode: 'onBlur'             — validate when field loses focus
// mode: 'onChange'           — validate on every keystroke (expensive)
// mode: 'onTouched'          — validate on first blur, then on change
// mode: 'all'                — validate on both change and blur

const { register } = useForm({ mode: 'onTouched' });
```

---

## 13. useFieldArray

`useFieldArray` manages **dynamic arrays** of fields — add/remove/reorder items.

```tsx
import { useForm, useFieldArray } from 'react-hook-form';

type FormValues = {
  skills: { name: string; level: string }[];
};

function SkillsForm() {
  const { register, control, handleSubmit } = useForm<FormValues>({
    defaultValues: {
      skills: [{ name: '', level: 'beginner' }],
    },
  });

  const { fields, append, remove, prepend, insert, move, update } = useFieldArray({
    control,
    name: 'skills',    // must match the array field name
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {fields.map((field, index) => (
        // IMPORTANT: use field.id as key — NOT index
        <div key={field.id}>
          <input
            {...register(`skills.${index}.name`, { required: true })}
            placeholder="Skill name"
          />
          <select {...register(`skills.${index}.level`)}>
            <option value="beginner">Beginner</option>
            <option value="advanced">Advanced</option>
          </select>
          <button type="button" onClick={() => remove(index)}>
            Remove
          </button>
        </div>
      ))}

      <button
        type="button"
        onClick={() => append({ name: '', level: 'beginner' })}
      >
        Add Skill
      </button>
      <button type="submit">Save</button>
    </form>
  );
}
```

### useFieldArray Methods

```ts
append(value)           // Add at end
prepend(value)          // Add at beginning
insert(index, value)    // Add at specific position
remove(index)           // Remove by index (or remove all if no arg)
move(from, to)          // Move item between positions
swap(indexA, indexB)    // Swap two items
update(index, value)    // Replace item at index
replace(newArray)       // Replace entire array
```

### Why field.id as Key

```tsx
// ❌ Wrong — using index as key breaks animations and React diffing when items are removed
{fields.map((field, index) => (
  <div key={index}>...</div>
))}

// ✅ Correct — each field has a stable id generated by useFieldArray
{fields.map((field, index) => (
  <div key={field.id}>...</div>
))}
```

---

## 14. Form Submission States

```tsx
function ContactForm() {
  const {
    register,
    handleSubmit,
    formState: { isSubmitting, isSubmitSuccessful, submitCount },
    reset,
  } = useForm<FormValues>();

  const [submitError, setSubmitError] = useState<string | null>(null);

  const onSubmit = async (data: FormValues) => {
    try {
      setSubmitError(null);
      await sendContactForm(data);
      reset();  // clear form on success
    } catch (err) {
      setSubmitError('Failed to send. Please try again.');
      // isSubmitSuccessful will be false because an error was thrown
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('message', { required: true })} />

      {submitError && <p className="error">{submitError}</p>}

      {isSubmitSuccessful && <p className="success">Message sent!</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Sending...' : `Send${submitCount > 0 ? ' Again' : ''}`}
      </button>
    </form>
  );
}
```

---

## 15. Performance Comparison

### Re-render Count Comparison

A form with 10 fields, user types 50 characters:

```text
Controlled (useState):
  10 fields × 50 keystrokes = 500 re-renders

Formik (controlled by default):
  Similar to controlled — ~500 re-renders

React Hook Form (uncontrolled):
  0 re-renders while typing
  Re-renders only for: errors, watch subscriptions, formState
```

### Bundle Size

```text
react-hook-form:  ~9 kB gzipped
Formik:          ~13 kB gzipped
```

### When Formik Has Advantages

- Simpler mental model for beginners
- Field-level components (Formik's `<Field>`) are ergonomic
- Arrays of objects are simpler (though RHF's `useFieldArray` is more powerful)

### RHF vs Formik vs Uncontrolled

| Dimension | RHF | Formik | Uncontrolled (plain) |
|---|---|---|---|
| Re-renders | Minimal | High | 0 |
| Validation | Built-in + resolvers | Yup by default | Manual |
| TypeScript | Excellent (inferred) | Good | None |
| Bundle size | Small | Medium | Zero |
| Dynamic arrays | useFieldArray | FieldArray | Manual |
| Controlled UI libs | Controller/useController | Custom Field | Ref |
| Learning curve | Moderate | Easy | None |

---

## 16. Common Mistakes

### Accessing formState Without Destructuring

```tsx
// ❌ Wrong — formState is a Proxy; accessing it without destructuring
//    prevents RHF from setting up subscriptions
const { formState } = useForm();
const isSubmitting = formState.isSubmitting;  // subscription not set up correctly

// ✅ Correct — destructure formState immediately
const { formState: { isSubmitting, errors } } = useForm();
// OR
const { formState } = useForm();
const { isSubmitting, errors } = formState;  // destructure before use
```

### Using index as useFieldArray Key

```tsx
// ❌ Wrong — breaks animations, React reconciliation on remove
{fields.map((field, index) => <div key={index}>...</div>)}

// ✅ Correct
{fields.map((field, index) => <div key={field.id}>...</div>)}
```

### Not Using reset() for Edit Forms

```tsx
// ❌ Wrong — setValue doesn't update defaultValues
//    isDirty will always be true even when form matches server data
useEffect(() => {
  if (data) {
    setValue('name', data.name);
    setValue('email', data.email);
  }
}, [data]);

// ✅ Correct — reset updates defaultValues too
useEffect(() => {
  if (data) {
    reset(data);  // sets values AND updates baseline for isDirty
  }
}, [data]);
```

### Forgetting to Unregister Dynamic Fields

```tsx
// ❌ When conditionally shown fields are unmounted, their values persist
// Use shouldUnregister: true to auto-remove unmounted fields
const { register } = useForm({ shouldUnregister: true });
```

---

## 17. Best Practices

### Use Zod for Schema-First Type Safety

Define your schema with Zod first — the TypeScript types are inferred automatically, and you get both client validation and type safety in one place.

```ts
const schema = z.object({ email: z.string().email(), age: z.number().min(18) });
type FormValues = z.infer<typeof schema>;  // type generated from schema
const { register } = useForm<FormValues>({ resolver: zodResolver(schema) });
```

### Use defaultValues for Every Field

Always provide `defaultValues` for all fields. Without defaults, fields start as `undefined` and RHF switches from uncontrolled to controlled mode — which can cause warnings.

### Prefer mode: 'onTouched'

`'onSubmit'` (default) shows no errors until submit. `'onChange'` shows errors on every keystroke. `'onTouched'` is the UX sweet spot — errors appear after a field has been visited and changed.

### Build Reusable Field Components with useController

Rather than spreading `Controller` everywhere, build typed reusable input components with `useController`. This keeps your forms clean and the validation logic encapsulated.

### Keep Large Forms in Sub-components with useFormContext

```tsx
// Pass form context via React Context — no prop drilling
import { FormProvider, useFormContext } from 'react-hook-form';

function ParentForm() {
  const methods = useForm<FormValues>();
  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        <PersonalSection />   {/* uses useFormContext */}
        <AddressSection />    {/* uses useFormContext */}
      </form>
    </FormProvider>
  );
}

function PersonalSection() {
  const { register, formState: { errors } } = useFormContext<FormValues>();
  // ...
}
```

### Summary Reference

| Concern | Recommendation |
|---|---|
| Validation | Zod + zodResolver for type-safe schemas |
| Validation mode | `'onTouched'` for best UX |
| Edit forms | Use `reset(serverData)` to set values AND baseline |
| Dynamic fields | `useFieldArray` with `field.id` as key |
| Controlled UI libs | `Controller` or `useController` |
| Large forms | `FormProvider` + `useFormContext` |
| Performance | Avoid `watch()` for fields not needing real-time feedback |
| TypeScript | Infer types with `z.infer<typeof schema>` |
| Defaults | Always specify `defaultValues` for all fields |
