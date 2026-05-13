## Machine Coding — Tricky Implementation Questions

> These questions test your ability to reason about component design decisions, identify implementation bugs, choose the right state structure, and predict the consequences of common patterns. Questions are in "What would happen if..." and "What is wrong with this implementation?" format.

---

## 1. Component Design

### Q1

```jsx
// Interviewer: Design a reusable Button component.
// Candidate's implementation:

function Button({ label, onClick, color, size, isLoading, disabled }) {
  return (
    <button
      onClick={onClick}
      disabled={disabled || isLoading}
      style={{
        backgroundColor: color || '#007bff',
        fontSize: size === 'sm' ? 12 : size === 'lg' ? 18 : 14,
      }}
    >
      {isLoading ? 'Loading...' : label}
    </button>
  );
}
```

#### ❓ What is wrong with this Button component API design? How would you improve it?

<details>
<summary>✅ Answer</summary>

```txt
Several design problems:

1. Accepts `label` as a prop — should use `children` for composability.
   Users cannot render icons or custom content inside the button.

2. `color` accepts any CSS color string — no design system enforcement.
   Better: accept `variant` ('primary' | 'secondary' | 'danger').

3. `size` uses magic strings without clear contract.
   Better: accept 'sm' | 'md' | 'lg' as a typed enum.

4. Does not forward other HTML button attributes (type, aria-*, data-*).
   Better: spread remaining props with ...rest.

5. 'Loading...' is hardcoded — not internationalizable.
```

Improved API:

```jsx
function Button({
  children,
  variant = 'primary',
  size = 'md',
  isLoading = false,
  loadingText,
  disabled,
  ...rest
}) {
  return (
    <button
      disabled={disabled || isLoading}
      aria-busy={isLoading}
      className={`btn btn--${variant} btn--${size}`}
      {...rest}
    >
      {isLoading ? (loadingText ?? children) : children}
    </button>
  );
}
```

</details>

---

### Q2

```jsx
function Tabs({ tabs }) {
  const [activeTab, setActiveTab] = useState(tabs[0].id);

  return (
    <div>
      <div role="tablist">
        {tabs.map(tab => (
          <button
            key={tab.id}
            role="tab"
            aria-selected={activeTab === tab.id}
            onClick={() => setActiveTab(tab.id)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      <div role="tabpanel">
        {tabs.find(t => t.id === activeTab)?.content}
      </div>
    </div>
  );
}
```

#### ❓ The `tabs` prop changes (new tabs are loaded). What problem can occur with `activeTab` state? How do you fix it?

<details>
<summary>✅ Answer</summary>

```txt
When `tabs` changes (e.g., different category of tabs loads), `activeTab`
still holds the old tab's ID. If the new tabs don't include an item with
that ID, `tabs.find(...)` returns undefined, and the tabpanel renders nothing.

The component shows no content without any indication of why.
```

Fix:

```jsx
// Sync activeTab when tabs prop changes
useEffect(() => {
  if (!tabs.find(t => t.id === activeTab)) {
    setActiveTab(tabs[0]?.id ?? null);
  }
}, [tabs]);
```

Or, use a controlled API where the parent manages `activeTab`:

```jsx
function Tabs({ tabs, activeTab, onTabChange }) { ... }
```

</details>

---

### Q3

```jsx
function StarRating({ value, onChange }) {
  return (
    <div>
      {[1, 2, 3, 4, 5].map(star => (
        <span
          key={star}
          onClick={() => onChange(star)}
          style={{ color: star <= value ? 'gold' : 'gray' }}
        >
          ★
        </span>
      ))}
    </div>
  );
}
```

#### ❓ This is a controlled component. What happens if `onChange` is not provided? How do you make it safe?

<details>
<summary>✅ Answer</summary>

```txt
When the user clicks a star, onChange(star) throws:
"TypeError: onChange is not a function"

The component crashes because onClick calls a prop that doesn't exist.
```

Fix:

```jsx
// Option 1: Default prop
function StarRating({ value = 0, onChange = () => {} }) { ... }

// Option 2: Optional chaining
onClick={() => onChange?.(star)}

// Option 3: Separate read-only mode
function StarRating({ value = 0, onChange, readOnly = false }) {
  return (
    <div>
      {[1,2,3,4,5].map(star => (
        <span
          key={star}
          onClick={readOnly ? undefined : () => onChange?.(star)}
          style={{
            cursor: readOnly ? 'default' : 'pointer',
            color: star <= value ? 'gold' : 'gray'
          }}
        >★</span>
      ))}
    </div>
  );
}
```

</details>

---

### Q4

```jsx
function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        {children}
      </div>
    </div>
  );
}
```

#### ❓ A user clicks inside the modal content and drags the mouse to the overlay before releasing. Does the modal close?

<details>
<summary>✅ Answer</summary>

```txt
Yes — the modal closes unexpectedly.

mousedown: inside .modal-content → stopPropagation does NOT apply here
mousemove: dragging to overlay
mouseup: fires on .modal-overlay → triggers onClick → onClose() is called

The user started the drag inside the modal but the click event fires
on the overlay, triggering unexpected close.
```

Fix: track mousedown target to distinguish a genuine click from a drag:

```jsx
function Modal({ isOpen, onClose, children }) {
  const mousedownTargetRef = useRef(null);

  if (!isOpen) return null;

  return (
    <div
      className="modal-overlay"
      onMouseDown={e => { mousedownTargetRef.current = e.target; }}
      onClick={e => {
        if (mousedownTargetRef.current === e.currentTarget) onClose();
      }}
    >
      <div className="modal-content">
        {children}
      </div>
    </div>
  );
}
```

</details>

---

### Q5

```jsx
// A toast notification system
function useToast() {
  const [toasts, setToasts] = useState([]);

  function addToast(message) {
    const id = Date.now();
    setToasts(prev => [...prev, { id, message }]);
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== id));
    }, 3000);
  }

  return { toasts, addToast };
}
```

#### ❓ The user clicks a button 5 times rapidly (within 1ms). What problem occurs with `Date.now()` as the ID?

<details>
<summary>✅ Answer</summary>

```txt
Two or more toasts may get the same ID if they are added within the
same millisecond. Date.now() has 1ms resolution.

Consequences:
1. Duplicate keys in the React list → React warning
2. The setTimeout filter `prev.filter(t => t.id !== id)` removes ALL
   toasts with that ID, not just the one it was supposed to dismiss.
   Multiple toasts disappear when only one should.
```

Fix: Use an incrementing counter or `crypto.randomUUID()`:

```jsx
let nextId = 1;

function addToast(message) {
  const id = nextId++;
  // or: const id = crypto.randomUUID();
  ...
}
```

</details>

---

## 2. State Management

### Q6

```jsx
function Accordion({ items }) {
  const [openItems, setOpenItems] = useState([]);

  function toggle(id) {
    setOpenItems(prev =>
      prev.includes(id)
        ? prev.filter(item => item !== id)
        : [...prev, id]
    );
  }

  return (
    <div>
      {items.map(item => (
        <AccordionItem
          key={item.id}
          item={item}
          isOpen={openItems.includes(item.id)}
          onToggle={() => toggle(item.id)}
        />
      ))}
    </div>
  );
}
```

#### ❓ The accordion has 10,000 items. What is the performance issue with `openItems.includes(id)` and `prev.filter(item => item !== id)`?

<details>
<summary>✅ Answer</summary>

```txt
Both operations are O(n) — they scan the entire openItems array.

1. openItems.includes(id): runs for every item on every render.
   With 10,000 items, this is 10,000 array scans per render.

2. prev.filter(item => item !== id): also O(n) on toggle.

Fix: Use a Set instead of an array.
Set.has() and Set.delete() are O(1):
```

```jsx
const [openItems, setOpenItems] = useState(new Set());

function toggle(id) {
  setOpenItems(prev => {
    const next = new Set(prev);
    if (next.has(id)) next.delete(id);
    else next.add(id);
    return next;
  });
}

// In render:
isOpen={openItems.has(item.id)}
```

</details>

---

### Q7

```jsx
function SearchFilter({ items }) {
  const [query, setQuery] = useState('');
  const [filtered, setFiltered] = useState(items);

  function handleSearch(e) {
    const value = e.target.value;
    setQuery(value);
    setFiltered(items.filter(item =>
      item.name.toLowerCase().includes(value.toLowerCase())
    ));
  }

  return (
    <>
      <input value={query} onChange={handleSearch} />
      <ul>{filtered.map(item => <li key={item.id}>{item.name}</li>)}</ul>
    </>
  );
}
```

#### ❓ What is wrong with storing `filtered` as state? What happens when `items` prop changes?

<details>
<summary>✅ Answer</summary>

```txt
Two problems:

1. Redundant state: `filtered` can always be derived from `query` and `items`.
   Storing derived values in state creates a state synchronization problem.

2. When `items` prop changes (e.g., new data loads), `filtered` is stale —
   it still shows the old list filtered by the current query.
   The two states are out of sync.
```

Fix: compute `filtered` during render (or with useMemo):

```jsx
function SearchFilter({ items }) {
  const [query, setQuery] = useState('');

  const filtered = useMemo(
    () => items.filter(item =>
      item.name.toLowerCase().includes(query.toLowerCase())
    ),
    [items, query]
  );

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <ul>{filtered.map(item => <li key={item.id}>{item.name}</li>)}</ul>
    </>
  );
}
```

</details>

---

### Q8

```jsx
function MultiStepForm() {
  const [step, setStep] = useState(1);
  const [step1Data, setStep1Data] = useState({ name: '', email: '' });
  const [step2Data, setStep2Data] = useState({ phone: '', address: '' });
  const [step3Data, setStep3Data] = useState({ payment: '', cvv: '' });

  async function submit() {
    await submitForm({ ...step1Data, ...step2Data, ...step3Data });
  }
}
```

#### ❓ What would happen if the user refreshes the page midway through the form? Should this be stored in a single object or separate state variables?

<details>
<summary>✅ Answer</summary>

```txt
Page refresh: All React state is lost. The user returns to step 1 with
empty fields. For critical multi-step flows (e.g., checkout), this is
a poor user experience.

Regarding single vs separate objects:
Using a single formData object is better here because:
1. All form data represents a single entity — a form submission
2. Easier to pass to the submit function: submitForm(formData)
3. Easier to persist to sessionStorage as a single JSON blob

For the persistence fix:
```

```jsx
const [formData, setFormData] = useSessionStorageState('multiStepForm', {
  name: '', email: '', phone: '', address: '', payment: ''
});
```

Or manually:

```jsx
const [formData, setFormData] = useState(() => {
  const saved = sessionStorage.getItem('formData');
  return saved ? JSON.parse(saved) : { name: '', email: '', phone: '' };
});

useEffect(() => {
  sessionStorage.setItem('formData', JSON.stringify(formData));
}, [formData]);
```

</details>

---

### Q9

```jsx
function PaginatedList() {
  const [currentPage, setCurrentPage] = useState(1);
  const [data, setData] = useState([]);

  useEffect(() => {
    fetchPage(currentPage).then(setData);
  }, [currentPage]);

  function goToPage(page) {
    setCurrentPage(page);
  }
}
```

#### ❓ The user is on page 3 and clicks "Back to Page 1". While the page-1 request is loading, they quickly click "Page 3" again. What could happen?

<details>
<summary>✅ Answer</summary>

```txt
Race condition:
1. Request for page 1 fires
2. User clicks page 3 — request for page 3 fires
3. Page 1 response arrives later → setData(page1Data) is called
4. UI shows page-3 URL/state but page-1 data

The user sees the wrong data for the current page.
```

Fix: use an ignore flag or AbortController in the effect:

```jsx
useEffect(() => {
  let ignore = false;

  setLoading(true);
  fetchPage(currentPage)
    .then(data => {
      if (!ignore) setData(data);
    })
    .finally(() => {
      if (!ignore) setLoading(false);
    });

  return () => { ignore = true; };
}, [currentPage]);
```

</details>

---

### Q10

```jsx
function DragList({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const [draggingId, setDraggingId] = useState(null);

  function handleDragStart(id) {
    setDraggingId(id);
  }

  function handleDrop(targetId) {
    const fromIndex = items.findIndex(i => i.id === draggingId);
    const toIndex = items.findIndex(i => i.id === targetId);
    const newItems = [...items];
    const [moved] = newItems.splice(fromIndex, 1);
    newItems.splice(toIndex, 0, moved);
    setItems(newItems);
    setDraggingId(null);
  }
}
```

#### ❓ `draggingId` is stored in `useState`. What is the problem with this, and what should it use instead?

<details>
<summary>✅ Answer</summary>

```txt
Every call to setDraggingId() triggers a re-render of the entire list.
During a drag, events fire rapidly (every pixel of movement triggers
onDragEnter). This causes continuous re-renders, making the drag
interaction janky, especially with long lists.

`draggingId` does not need to cause re-renders — it's read only
at the moment of drop, not during render.

Fix: use useRef instead of useState:
```

```jsx
const draggingIdRef = useRef(null);

function handleDragStart(id) {
  draggingIdRef.current = id; // no re-render
}

function handleDrop(targetId) {
  const fromIndex = items.findIndex(i => i.id === draggingIdRef.current);
  const toIndex = items.findIndex(i => i.id === targetId);
  const newItems = [...items];
  const [moved] = newItems.splice(fromIndex, 1);
  newItems.splice(toIndex, 0, moved);
  setItems(newItems); // only one re-render, on drop
  draggingIdRef.current = null;
}
```

</details>

---

## 3. Performance

### Q11

```jsx
function ProductList({ products, onAddToCart }) {
  return (
    <ul>
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onAddToCart={() => onAddToCart(product.id)} // inline arrow
        />
      ))}
    </ul>
  );
}

const ProductCard = React.memo(function ProductCard({ product, onAddToCart }) {
  console.log('ProductCard rendered:', product.name);
  return (
    <li>
      {product.name}
      <button onClick={onAddToCart}>Add to cart</button>
    </li>
  );
});
```

#### ❓ There are 100 products. The cart state updates (an unrelated state in the parent). Do all 100 ProductCards re-render?

<details>
<summary>✅ Answer</summary>

```txt
Yes — all 100 ProductCards re-render.
```

**Explanation:** `() => onAddToCart(product.id)` is a new arrow function created for each product on every render of `ProductList`. `React.memo` compares `prevProps.onAddToCart` with `nextProps.onAddToCart` via `Object.is` — different function references → re-render. `React.memo` is defeated by the inline arrow function.

Fix: pass `product.id` as a prop and let `ProductCard` call `onAddToCart(product.id)` directly, where `onAddToCart` is stabilized with `useCallback`:

```jsx
<ProductCard
  key={product.id}
  product={product}
  productId={product.id}
  onAddToCart={onAddToCart} // stable reference from useCallback
/>
```

</details>

---

### Q12

```jsx
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState([]);

  // No debounce
  useEffect(() => {
    if (query) fetchResults(query).then(setResults);
  }, [query]);

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Search..."
    />
  );
}
```

#### ❓ The user types "react hooks" (10 characters). How many API requests fire? What is the impact?

<details>
<summary>✅ Answer</summary>

```txt
10 API requests fire — one per character: "r", "re", "rea", "reac", etc.

Impact:
1. Server receives 10 requests, processes all 10, sends 10 responses
2. Network bandwidth wasted on requests that are immediately superseded
3. Race condition risk: the response for "r" (fast, short query) may
   arrive after the response for "react" (slower, more data), overwriting
   the correct results with stale data
4. Input may feel sluggish if each network round-trip blocks rendering
```

Fix: Add a debounce of 300ms — only fires when the user pauses typing.

</details>

---

### Q13

```jsx
function CommentThread({ comments }) {
  return (
    <div>
      {comments.map(comment => (
        <Comment key={comment.id} comment={comment} />
      ))}
    </div>
  );
}

function Comment({ comment }) {
  const formattedDate = new Date(comment.createdAt).toLocaleDateString('en-IN', {
    year: 'numeric', month: 'long', day: 'numeric'
  });

  return (
    <div>
      <p>{comment.text}</p>
      <span>{formattedDate}</span>
    </div>
  );
}
```

#### ❓ There are 500 comments and the parent re-renders frequently. Should `formattedDate` be memoized? What is the trade-off?

<details>
<summary>✅ Answer</summary>

```txt
It depends on the measurement. `toLocaleDateString` is not free —
locale-aware date formatting involves the Intl API. For 500 comments
re-rendering frequently, this could add up.

However, useMemo has its own overhead (closure creation, dependency comparison).
For a simple date formatting operation, the useMemo overhead may exceed the savings.

The most impactful optimization is:
1. Wrap Comment in React.memo — if comment prop hasn't changed, skip entirely
2. Only then consider memoizing formattedDate if Profiler shows it as a bottleneck

Profile first, optimize second.
```

```jsx
const Comment = React.memo(function Comment({ comment }) {
  // With React.memo, this only runs when comment changes
  const formattedDate = new Date(comment.createdAt).toLocaleDateString('en-IN', {
    year: 'numeric', month: 'long', day: 'numeric'
  });
  // ...
});
```

</details>

---

### Q14

```jsx
function FileList({ files }) {
  return (
    <div style={{ height: '600px', overflow: 'auto' }}>
      {files.map(file => (
        <FileRow key={file.id} file={file} />
      ))}
    </div>
  );
}
```

#### ❓ `files` contains 50,000 items. A user scrolls through the list. What performance problem occurs and how do you solve it?

<details>
<summary>✅ Answer</summary>

```txt
50,000 DOM nodes are created and inserted into the document.

Problems:
1. Initial render takes several seconds (creating 50,000 DOM nodes)
2. Memory usage is high (DOM nodes, React fiber nodes for all 50,000)
3. Scrolling is janky because the browser must maintain layout for all nodes
4. Any parent re-render triggers reconciliation of all 50,000 items

Fix: Virtualize the list — only render the items visible in the viewport.
```

```jsx
import { useVirtualizer } from '@tanstack/react-virtual';

function FileList({ files }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: files.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48, // each row is ~48px
    overscan: 10,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(vItem => (
          <div
            key={vItem.key}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: vItem.size,
              transform: `translateY(${vItem.start}px)`,
            }}
          >
            <FileRow file={files[vItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

Only ~15 DOM nodes exist at any time regardless of list size.

</details>

---

### Q15

```jsx
function AutocompleteInput({ fetchSuggestions }) {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState([]);

  const debouncedFetch = useMemo(
    () => debounce((q) => {
      fetchSuggestions(q).then(setSuggestions);
    }, 300),
    [] // empty deps
  );

  useEffect(() => {
    if (query) debouncedFetch(query);
  }, [query, debouncedFetch]);
}
```

#### ❓ `fetchSuggestions` prop changes (parent passes a new function). What problem occurs?

<details>
<summary>✅ Answer</summary>

```txt
`debouncedFetch` is memoized with `[]` (no deps). It captures the original
`fetchSuggestions` from mount via closure. When the parent passes a new
`fetchSuggestions`, the debounced function still calls the original version.

The component is stale — it never uses the updated fetchSuggestions.
```

Fix: add `fetchSuggestions` to useMemo's deps:

```jsx
const debouncedFetch = useMemo(
  () => debounce((q) => {
    fetchSuggestions(q).then(setSuggestions);
  }, 300),
  [fetchSuggestions] // recreate when fetchSuggestions changes
);
```

But note: recreating the debounced function resets the debounce timer.
If the parent stabilizes `fetchSuggestions` with `useCallback`, this is not an issue.

</details>

---

## 4. Edge Cases

### Q16

```jsx
function InfiniteScrollFeed() {
  const [posts, setPosts] = useState([]);
  const [page, setPage] = useState(1);

  useEffect(() => {
    fetchPosts(page).then(data => {
      setPosts(prev => [...prev, ...data.posts]);
    });
  }, [page]);

  return (
    <>
      {posts.map(post => <PostCard key={post.id} post={post} />)}
      <button onClick={() => setPage(p => p + 1)}>Load More</button>
    </>
  );
}
```

#### ❓ What happens if `fetchPosts` returns no items (end of feed)? What should the component do?

<details>
<summary>✅ Answer</summary>

```txt
Currently: The "Load More" button stays visible and functional.
Clicking it fires a request that returns empty data. The posts list
doesn't change but an unnecessary API call is made. Repeated clicking
sends repeated empty requests.
```

Fix: track whether more data exists:

```jsx
const [hasMore, setHasMore] = useState(true);

useEffect(() => {
  fetchPosts(page).then(data => {
    setPosts(prev => [...prev, ...data.posts]);
    setHasMore(data.hasNextPage); // server tells us if more pages exist
  });
}, [page]);

return (
  <>
    {posts.length === 0 && <p>No posts yet.</p>}
    {posts.map(post => <PostCard key={post.id} post={post} />)}
    {hasMore ? (
      <button onClick={() => setPage(p => p + 1)}>Load More</button>
    ) : (
      <p>You've reached the end.</p>
    )}
  </>
);
```

</details>

---

### Q17

```jsx
function FileUpload() {
  const [files, setFiles] = useState([]);

  function handleChange(e) {
    setFiles(Array.from(e.target.files));
  }

  function removeFile(index) {
    setFiles(prev => prev.filter((_, i) => i !== index));
  }

  return (
    <>
      <input type="file" multiple onChange={handleChange} />
      {files.map((file, index) => (
        <div key={index}>
          {file.name}
          <button onClick={() => removeFile(index)}>Remove</button>
        </div>
      ))}
    </>
  );
}
```

#### ❓ The user selects 3 files and removes the middle one. Then selects 3 more files using the same input. What happens to the list?

<details>
<summary>✅ Answer</summary>

```txt
Problem 1: When the user selects new files via the input, handleChange
sets `files` to ONLY the newly selected files, discarding the previous
selection. The original 2 remaining files are gone.

Problem 2: key={index} — after removing the middle file, the remaining
files' indices shift. React reconciles incorrectly, potentially showing
the wrong file names briefly.

Fix: use file name + lastModified as key, and append new files:
```

```jsx
function handleChange(e) {
  const newFiles = Array.from(e.target.files);
  setFiles(prev => {
    // Deduplicate by name
    const existingNames = new Set(prev.map(f => f.name));
    return [...prev, ...newFiles.filter(f => !existingNames.has(f.name))];
  });
  e.target.value = ''; // reset input so same file can be re-added
}

// Key using stable file identity
{files.map(file => (
  <div key={`${file.name}-${file.lastModified}`}>
    ...
  </div>
))}
```

</details>

---

### Q18

```jsx
function SearchBar({ onSearch }) {
  const [query, setQuery] = useState('');

  return (
    <form onSubmit={e => { e.preventDefault(); onSearch(query); }}>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <button type="submit">Search</button>
    </form>
  );
}
```

#### ❓ The user searches for "react", gets results, then clears the query and searches with an empty string. What should happen? Does this implementation handle it correctly?

<details>
<summary>✅ Answer</summary>

```txt
Current behavior: onSearch("") is called, which likely fetches all results
or shows an unfiltered list — possibly not the intended behavior.

The component should handle two cases:
1. Empty query on submit: either do nothing or reset to default state
2. Show a "clear" button that resets both query and results

The implementation does not guard against empty submit:
```

```jsx
<form onSubmit={e => {
  e.preventDefault();
  if (!query.trim()) return; // guard against empty search
  onSearch(query.trim());
}}>

// Or: provide explicit clear behavior
function handleClear() {
  setQuery('');
  onSearch(''); // signal parent to show default state
}
```

The right behavior depends on requirements — clarify with the interviewer.

</details>

---

### Q19

```jsx
function RatingWidget({ onRate }) {
  const [rating, setRating] = useState(0);
  const [submitted, setSubmitted] = useState(false);

  function handleRate(value) {
    setRating(value);
  }

  function handleSubmit() {
    onRate(rating);
    setSubmitted(true);
  }

  if (submitted) return <p>Thank you for your rating!</p>;

  return (
    <div>
      <StarRating value={rating} onChange={handleRate} />
      <button onClick={handleSubmit} disabled={rating === 0}>
        Submit Rating
      </button>
    </div>
  );
}
```

#### ❓ What happens if `onRate` throws an error (e.g., API call fails)? How do you handle this gracefully?

<details>
<summary>✅ Answer</summary>

```txt
Currently: If onRate throws synchronously or rejects (if async),
the error is unhandled. setSubmitted(true) may have already been
called (if the error is asynchronous) — showing "Thank you!" even
though the rating wasn't actually saved.
```

Fix: make onRate async and handle the error:

```jsx
const [error, setError] = useState(null);
const [isSubmitting, setIsSubmitting] = useState(false);

async function handleSubmit() {
  setIsSubmitting(true);
  setError(null);
  try {
    await onRate(rating);
    setSubmitted(true);
  } catch (err) {
    setError('Failed to submit rating. Please try again.');
  } finally {
    setIsSubmitting(false);
  }
}

// In render:
{error && <p style={{ color: 'red' }}>{error}</p>}
<button onClick={handleSubmit} disabled={rating === 0 || isSubmitting}>
  {isSubmitting ? 'Submitting...' : 'Submit Rating'}
</button>
```

</details>

---

### Q20

```jsx
function Pagination({ totalItems, pageSize, currentPage, onPageChange }) {
  const totalPages = Math.ceil(totalItems / pageSize);

  return (
    <div>
      {Array.from({ length: totalPages }, (_, i) => (
        <button key={i + 1} onClick={() => onPageChange(i + 1)}>
          {i + 1}
        </button>
      ))}
    </div>
  );
}
```

#### ❓ `totalItems` is 0. What renders? What about `totalItems = 1` with `pageSize = 10`?

<details>
<summary>✅ Answer</summary>

```txt
totalItems = 0:
  Math.ceil(0 / 10) = 0
  Array.from({ length: 0 }) = []
  Renders nothing — no buttons, no indication that there are zero pages.
  This is technically correct but may confuse users.

totalItems = 1, pageSize = 10:
  Math.ceil(1 / 10) = 1
  Renders one button: "1"
  Correct — a single page for 1 item.

Edge case to handle:
  If totalPages <= 1, consider hiding pagination entirely:
```

```jsx
if (totalPages <= 1) return null;
```

</details>

---

## 5. Implementation Patterns

### Q21

```jsx
function Toggle({ defaultOn = false }) {
  const [on, setOn] = useState(defaultOn);

  return (
    <button onClick={() => setOn(prev => !prev)}>
      {on ? 'ON' : 'OFF'}
    </button>
  );
}
```

#### ❓ The parent wants to control the toggle programmatically (e.g., reset all toggles on a button click). How would you extend this component to support both uncontrolled and controlled usage?

<details>
<summary>✅ Answer</summary>

```txt
This is the "controlled vs uncontrolled" pattern.
To support both:
1. If `value` and `onChange` props are provided → controlled mode
2. If not provided → uncontrolled mode (use internal state)
```

```jsx
function Toggle({ value, onChange, defaultOn = false }) {
  // Internal state for uncontrolled mode
  const [internalOn, setInternalOn] = useState(defaultOn);

  // Determine if controlled
  const isControlled = value !== undefined;
  const isOn = isControlled ? value : internalOn;

  function handleToggle() {
    if (isControlled) {
      onChange?.(!value);
    } else {
      setInternalOn(prev => !prev);
      onChange?.(!internalOn);
    }
  }

  return (
    <button onClick={handleToggle}>
      {isOn ? 'ON' : 'OFF'}
    </button>
  );
}

// Uncontrolled usage:
<Toggle defaultOn={true} />

// Controlled usage:
<Toggle value={isFeatureEnabled} onChange={setIsFeatureEnabled} />
```

</details>

---

### Q22

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

#### ❓ What is wrong with this debounce hook when `delay` changes during usage?

<details>
<summary>✅ Answer</summary>

```txt
When `delay` changes, the effect re-runs:
1. The previous timer is cleared
2. A new timer is set with the new delay

This is actually correct behavior — changing the delay mid-use should
reset the timer. However there is a subtle issue:

If `delay` is passed as a literal (e.g., useDebounce(query, 300)),
it is a stable primitive and won't cause unexpected re-runs.

But if `delay` is a computed value or comes from state and changes
frequently, the debounce timer resets on every delay change, potentially
causing the debounced value to never update.

Best practice: pass `delay` as a constant, not a dynamic value.
If the delay must change, use a ref for it to avoid triggering the effect:
```

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);
  const delayRef = useRef(delay);
  delayRef.current = delay; // always current without triggering effect

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delayRef.current);

    return () => clearTimeout(timer);
  }, [value]); // only re-run when value changes

  return debouncedValue;
}
```

</details>

---

### Q23

```jsx
function Autocomplete() {
  const [query, setQuery] = useState('');
  const [isOpen, setIsOpen] = useState(false);
  const [suggestions, setSuggestions] = useState([]);

  return (
    <div>
      <input
        value={query}
        onChange={e => { setQuery(e.target.value); setIsOpen(true); }}
        onBlur={() => setIsOpen(false)}
      />
      {isOpen && (
        <ul>
          {suggestions.map(s => (
            <li key={s.id} onClick={() => setQuery(s.label)}>
              {s.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

#### ❓ The user clicks a suggestion. The suggestion does not get selected. What is the bug and what is the fix?

<details>
<summary>✅ Answer</summary>

```txt
Bug: onBlur fires BEFORE onClick.

Event order when user clicks a suggestion:
1. mousedown on <li>
2. blur fires on <input> → setIsOpen(false) → dropdown unmounts
3. The <li> is removed from DOM
4. click event was supposed to fire on the <li> — but it no longer exists
5. The onClick handler never runs

The suggestion is never selected.
```

Fix option 1: Use `onMouseDown` with `e.preventDefault()` on suggestion items:

```jsx
<li
  key={s.id}
  onMouseDown={e => {
    e.preventDefault(); // prevent input blur
    setQuery(s.label);
    setIsOpen(false);
  }}
>
```

Fix option 2: Delay closing the dropdown:

```jsx
onBlur={() => setTimeout(() => setIsOpen(false), 150)}
```

Option 1 is cleaner and more reliable.

</details>

---

### Q24

```jsx
function ProgressStepper({ steps, currentStep }) {
  return (
    <div style={{ display: 'flex' }}>
      {steps.map((step, index) => (
        <div
          key={step.id}
          style={{
            flex: 1,
            textAlign: 'center',
            opacity: index > currentStep ? 0.4 : 1,
          }}
        >
          <div
            style={{
              width: 32,
              height: 32,
              borderRadius: '50%',
              background: index < currentStep ? '#4caf50' :
                          index === currentStep ? '#2196f3' : '#e0e0e0',
              margin: '0 auto',
              lineHeight: '32px',
            }}
          >
            {index < currentStep ? '✓' : index + 1}
          </div>
          <p>{step.label}</p>
        </div>
      ))}
    </div>
  );
}
```

#### ❓ What is the accessibility issue with this progress stepper?

<details>
<summary>✅ Answer</summary>

```txt
Multiple accessibility issues:

1. No screen reader information about the current step or progress.
   A visually impaired user has no way to know which step they are on.

2. The checkmark '✓' and step numbers are inside a styled div with
   no semantic meaning.

3. No role="progressbar" or aria attributes to communicate state.
```

Fix:

```jsx
<div
  role="list"
  aria-label="Progress steps"
  style={{ display: 'flex' }}
>
  {steps.map((step, index) => (
    <div
      key={step.id}
      role="listitem"
      aria-current={index === currentStep ? 'step' : undefined}
      aria-label={
        index < currentStep ? `${step.label}: completed` :
        index === currentStep ? `${step.label}: current step` :
        `${step.label}: not started`
      }
      style={{ flex: 1, textAlign: 'center' }}
    >
      ...
    </div>
  ))}
</div>
```

</details>

---

### Q25

```jsx
function NotificationBadge({ count }) {
  return (
    <div style={{ position: 'relative', display: 'inline-block' }}>
      <BellIcon />
      {count > 0 && (
        <span
          style={{
            position: 'absolute',
            top: -4,
            right: -4,
            background: 'red',
            color: 'white',
            borderRadius: '50%',
            width: 20,
            height: 20,
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            fontSize: 11,
          }}
        >
          {count}
        </span>
      )}
    </div>
  );
}
```

#### ❓ `count` is 1,500. What renders in the badge? How would you improve the display for large numbers?

<details>
<summary>✅ Answer</summary>

```txt
"1500" renders in the badge. The number doesn't fit in the 20x20 circle,
overflowing or being clipped.

Typical patterns for large notification counts:
1. Cap at 99+: display "99+" for counts > 99
2. Cap at 9+ for tighter spaces
3. Use "1k", "1.5k" for very large counts
```

```jsx
function formatCount(count) {
  if (count > 999) return '999+';
  if (count > 99) return '99+';
  return String(count);
}

// In component:
{count > 0 && (
  <span
    style={{
      ...badgeStyles,
      minWidth: 20,  // use minWidth instead of fixed width
      padding: '0 4px',
    }}
    aria-label={`${count} unread notifications`}
  >
    {formatCount(count)}
  </span>
)}
```

</details>

---

## Topics Covered

| Category | Questions | Key Concepts |
|---|---|---|
| Component Design | Q1 – Q5 | Button API design, tabs with changing data, controlled component safety, modal drag vs click, Date.now() ID collision |
| State Management | Q6 – Q10 | Set vs array for toggle state, derived state anti-pattern, single object for wizard forms, race condition in pagination, useRef vs useState for drag |
| Performance | Q11 – Q15 | Inline arrow defeating React.memo, unbounced search requests, useMemo vs React.memo decision, virtual list for large data, stale closure in debounce useMemo |
| Edge Cases | Q16 – Q20 | End-of-feed detection, file input reset, empty search guard, error handling in rating submit, zero-item pagination |
| Implementation Patterns | Q21 – Q25 | Controlled vs uncontrolled toggle, delay-ref pattern in debounce, blur-before-click in autocomplete, accessible progress stepper, notification badge overflow |
