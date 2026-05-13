# React Machine Coding

## Table of Contents

- [1. What Machine Coding Rounds Test](#1-what-machine-coding-rounds-test)
- [2. General Implementation Approach](#2-general-implementation-approach)
- [3. Star Rating Component](#3-star-rating-component)
- [4. Infinite Scroll](#4-infinite-scroll)
- [5. Search with Debounce and Autocomplete](#5-search-with-debounce-and-autocomplete)
- [6. Pagination Component](#6-pagination-component)
- [7. Drag and Drop List](#7-drag-and-drop-list)
- [8. Accordion and Collapsible](#8-accordion-and-collapsible)
- [9. Modal and Dialog](#9-modal-and-dialog)
- [10. Toast and Notification System](#10-toast-and-notification-system)
- [11. Form with Validation](#11-form-with-validation)
- [12. File Upload with Preview](#12-file-upload-with-preview)
- [13. Real-Time Search Filter](#13-real-time-search-filter)
- [14. Multi-Step Form and Wizard](#14-multi-step-form-and-wizard)
- [15. Common Pitfalls in Machine Coding Rounds](#15-common-pitfalls-in-machine-coding-rounds)
- [16. Performance in Machine Coding](#16-performance-in-machine-coding)
- [17. Testing Your Implementation](#17-testing-your-implementation)
- [18. Code Organization Patterns](#18-code-organization-patterns)
- [19. Best Practices for Live Coding](#19-best-practices-for-live-coding)

---

## 1. What Machine Coding Rounds Test

Machine coding rounds evaluate how you build real features under time pressure. Interviewers assess:

- **Practical implementation skill** — can you write working code, not just theory?
- **Component design** — do you break problems into clean, composable pieces?
- **State management** — do you know what state to keep, where to put it, and how to update it?
- **Edge case handling** — do you think about empty states, loading states, and error states?
- **Code quality** — is the code readable, well-named, and maintainable?
- **Optimization instinct** — do you know when to debounce, memoize, or virtualize?

The bar is working code with clean design, not perfect code. Communicate your thinking as you build.

---

## 2. General Implementation Approach

Follow this 5-step process for any machine coding problem:

```text
Step 1: Clarify requirements
  - What does the component receive as props?
  - What should it emit (events, callbacks)?
  - What are the edge cases? (empty list, single item, max value)
  - Is there a controlled vs uncontrolled API?

Step 2: Identify state
  - What data changes over time?
  - Where does the state live? (local vs lifted vs context)
  - What can be derived from existing state? (avoid redundant state)

Step 3: Sketch component structure
  - What are the sub-components?
  - What props does each receive?
  - Where do callbacks live?

Step 4: Build the happy path first
  - Make it work for the simple case
  - Then add edge cases
  - Then optimize if asked

Step 5: Handle edge cases
  - Empty state
  - Loading state
  - Error state
  - Boundary conditions (first/last item, max/min values)
```

---

## 3. Star Rating Component

---

### Requirements

- Render N stars (default 5)
- Click to set rating
- Hover to preview rating
- Support read-only mode
- Callback for rating change

---

### Implementation

```jsx
import { useState } from 'react';

function StarRating({
  maxRating = 5,
  initialRating = 0,
  readOnly = false,
  onRatingChange,
  size = 24,
}) {
  const [rating, setRating] = useState(initialRating);
  const [hovered, setHovered] = useState(0);

  function handleClick(value) {
    if (readOnly) return;
    setRating(value);
    onRatingChange?.(value);
  }

  const displayed = hovered || rating;

  return (
    <div
      style={{ display: 'flex', gap: 4 }}
      onMouseLeave={() => setHovered(0)}
      role="group"
      aria-label={`Rating: ${rating} out of ${maxRating}`}
    >
      {Array.from({ length: maxRating }, (_, i) => {
        const value = i + 1;
        const isFilled = value <= displayed;

        return (
          <span
            key={value}
            onClick={() => handleClick(value)}
            onMouseEnter={() => !readOnly && setHovered(value)}
            style={{
              fontSize: size,
              cursor: readOnly ? 'default' : 'pointer',
              color: isFilled ? '#f5a623' : '#ccc',
              transition: 'color 0.1s',
            }}
            role={readOnly ? 'img' : 'button'}
            aria-label={`${value} star${value > 1 ? 's' : ''}`}
          >
            ★
          </span>
        );
      })}
    </div>
  );
}
```

---

### Key State Decisions

| State | Why local | What it drives |
|---|---|---|
| `rating` | Controlled within this component | Filled stars |
| `hovered` | Transient UI state — no need to share | Preview on mouse hover |

The component is **uncontrolled by default** (manages its own `rating`) but exposes `onRatingChange` for parent integration.

---

## 4. Infinite Scroll

---

### Requirements

- Fetch items on demand as user scrolls down
- Show loader while fetching
- Stop fetching when no more items exist
- Handle fetch errors

---

### Implementation with IntersectionObserver

```jsx
import { useState, useEffect, useRef, useCallback } from 'react';

function InfiniteScrollList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const sentinelRef = useRef(null);

  const fetchItems = useCallback(async (pageNum) => {
    if (loading || !hasMore) return;

    setLoading(true);
    try {
      const data = await fetchPage(pageNum);
      setItems(prev => [...prev, ...data.items]);
      setHasMore(data.hasNextPage);
    } catch (err) {
      console.error('Failed to fetch:', err);
    } finally {
      setLoading(false);
    }
  }, [loading, hasMore]);

  // Fetch on page change
  useEffect(() => {
    fetchItems(page);
  }, [page]);

  // Observe sentinel element
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasMore && !loading) {
          setPage(prev => prev + 1);
        }
      },
      { rootMargin: '200px' } // preload 200px before reaching bottom
    );

    if (sentinelRef.current) {
      observer.observe(sentinelRef.current);
    }

    return () => observer.disconnect();
  }, [hasMore, loading]);

  return (
    <div>
      <ul>
        {items.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
      {loading && <div>Loading more...</div>}
      {!hasMore && <div>No more items</div>}
      <div ref={sentinelRef} style={{ height: 1 }} />
    </div>
  );
}
```

---

### Common Mistake

Forgetting to guard against concurrent fetches — if the user scrolls fast, `fetchItems` can be called while a previous call is still in flight. The `loading` guard prevents double-fetching.

---

## 5. Search with Debounce and Autocomplete

---

### Requirements

- Input triggers search suggestions
- Debounce API calls (300ms)
- Show dropdown of results
- Handle keyboard navigation (↑↓ Enter Escape)
- Handle click vs blur conflict

---

### useDebounce Hook

```jsx
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

---

### Autocomplete Component

```jsx
function Autocomplete({ onSelect }) {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState([]);
  const [selectedIndex, setSelectedIndex] = useState(-1);
  const [isOpen, setIsOpen] = useState(false);
  const [loading, setLoading] = useState(false);
  const debouncedQuery = useDebounce(query, 300);

  // Fetch suggestions when debounced query changes
  useEffect(() => {
    if (!debouncedQuery.trim()) {
      setSuggestions([]);
      setIsOpen(false);
      return;
    }

    let cancelled = false;
    setLoading(true);

    fetchSuggestions(debouncedQuery)
      .then(data => {
        if (!cancelled) {
          setSuggestions(data);
          setIsOpen(true);
          setSelectedIndex(-1);
        }
      })
      .finally(() => {
        if (!cancelled) setLoading(false);
      });

    return () => { cancelled = true; };
  }, [debouncedQuery]);

  function handleKeyDown(e) {
    if (!isOpen) return;

    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setSelectedIndex(prev =>
          prev < suggestions.length - 1 ? prev + 1 : 0
        );
        break;
      case 'ArrowUp':
        e.preventDefault();
        setSelectedIndex(prev =>
          prev > 0 ? prev - 1 : suggestions.length - 1
        );
        break;
      case 'Enter':
        if (selectedIndex >= 0) {
          handleSelect(suggestions[selectedIndex]);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        setSelectedIndex(-1);
        break;
    }
  }

  function handleSelect(suggestion) {
    setQuery(suggestion.label);
    setIsOpen(false);
    onSelect?.(suggestion);
  }

  return (
    <div style={{ position: 'relative' }}>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        onKeyDown={handleKeyDown}
        onBlur={() => setTimeout(() => setIsOpen(false), 150)}
        onFocus={() => suggestions.length > 0 && setIsOpen(true)}
        aria-autocomplete="list"
        aria-expanded={isOpen}
      />
      {loading && <span>Loading...</span>}
      {isOpen && suggestions.length > 0 && (
        <ul
          style={{
            position: 'absolute',
            top: '100%',
            left: 0,
            right: 0,
            background: 'white',
            border: '1px solid #ccc',
            listStyle: 'none',
            margin: 0,
            padding: 0,
            zIndex: 1000,
          }}
        >
          {suggestions.map((s, index) => (
            <li
              key={s.id}
              onMouseDown={e => { e.preventDefault(); handleSelect(s); }}
              style={{
                padding: '8px 12px',
                background: index === selectedIndex ? '#e8f0fe' : 'white',
                cursor: 'pointer',
              }}
            >
              {s.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

### Critical Detail: onMouseDown vs onClick

Using `onClick` on suggestions causes a bug: `onBlur` fires before `onClick`, hiding the dropdown before the click registers. `onMouseDown` fires before `onBlur`. Calling `e.preventDefault()` prevents the input from losing focus, keeping the dropdown visible so the click completes.

---

## 6. Pagination Component

---

### Requirements

- Show page numbers with prev/next buttons
- Handle first and last page boundaries
- Show ellipsis for large page counts
- Callback when page changes

---

### Page Number Generation

```jsx
function getPageNumbers(currentPage, totalPages, delta = 2) {
  const range = [];
  const rangeWithDots = [];

  for (
    let i = Math.max(1, currentPage - delta);
    i <= Math.min(totalPages, currentPage + delta);
    i++
  ) {
    range.push(i);
  }

  if (range[0] > 1) {
    rangeWithDots.push(1);
    if (range[0] > 2) rangeWithDots.push('...');
  }

  rangeWithDots.push(...range);

  if (range[range.length - 1] < totalPages) {
    if (range[range.length - 1] < totalPages - 1) rangeWithDots.push('...');
    rangeWithDots.push(totalPages);
  }

  return rangeWithDots;
}
```

---

### Pagination Component

```jsx
function Pagination({ currentPage, totalPages, onPageChange }) {
  const pages = getPageNumbers(currentPage, totalPages);

  return (
    <nav aria-label="Pagination">
      <button
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage === 1}
        aria-label="Previous page"
      >
        Previous
      </button>

      {pages.map((page, index) =>
        page === '...' ? (
          <span key={`ellipsis-${index}`} aria-hidden="true">...</span>
        ) : (
          <button
            key={page}
            onClick={() => onPageChange(page)}
            aria-current={page === currentPage ? 'page' : undefined}
            style={{
              fontWeight: page === currentPage ? 'bold' : 'normal',
            }}
          >
            {page}
          </button>
        )
      )}

      <button
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage === totalPages}
        aria-label="Next page"
      >
        Next
      </button>
    </nav>
  );
}
```

---

## 7. Drag and Drop List

---

### Requirements

- Reorder items by dragging
- Visual feedback during drag
- Update order in state on drop

---

### Implementation with HTML5 Drag API

```jsx
function DraggableList({ initialItems }) {
  const [items, setItems] = useState(initialItems);
  const dragItem = useRef(null);
  const dragOverItem = useRef(null);

  function handleDragStart(index) {
    dragItem.current = index;
  }

  function handleDragEnter(index) {
    dragOverItem.current = index;
  }

  function handleDrop() {
    const newItems = [...items];
    const dragged = newItems.splice(dragItem.current, 1)[0];
    newItems.splice(dragOverItem.current, 0, dragged);
    dragItem.current = null;
    dragOverItem.current = null;
    setItems(newItems);
  }

  return (
    <ul style={{ listStyle: 'none', padding: 0 }}>
      {items.map((item, index) => (
        <li
          key={item.id}
          draggable
          onDragStart={() => handleDragStart(index)}
          onDragEnter={() => handleDragEnter(index)}
          onDragEnd={handleDrop}
          onDragOver={e => e.preventDefault()}
          style={{
            padding: '12px 16px',
            marginBottom: 8,
            background: 'white',
            border: '1px solid #e0e0e0',
            borderRadius: 4,
            cursor: 'grab',
            userSelect: 'none',
          }}
        >
          ⠿ {item.name}
        </li>
      ))}
    </ul>
  );
}
```

---

### State Design for Drag

Drag state uses `useRef` rather than `useState` because:
- Drag state is transient — not needed for rendering during drag
- `useRef` updates do not trigger re-renders (avoids jank during drag)
- Only the final `drop` event updates `items` state (causing one re-render)

---

## 8. Accordion and Collapsible

---

### Requirements

- Click to expand/collapse sections
- Support single open or multiple open at once
- Animated expand/collapse

---

### Single Open Accordion

```jsx
function Accordion({ items }) {
  const [openIndex, setOpenIndex] = useState(null);

  function toggle(index) {
    setOpenIndex(prev => prev === index ? null : index);
  }

  return (
    <div>
      {items.map((item, index) => (
        <div key={item.id} style={{ borderBottom: '1px solid #e0e0e0' }}>
          <button
            onClick={() => toggle(index)}
            style={{
              width: '100%',
              textAlign: 'left',
              padding: '16px',
              background: 'none',
              border: 'none',
              cursor: 'pointer',
              fontWeight: 600,
              display: 'flex',
              justifyContent: 'space-between',
            }}
            aria-expanded={openIndex === index}
            aria-controls={`panel-${index}`}
          >
            {item.title}
            <span>{openIndex === index ? '▲' : '▼'}</span>
          </button>
          <div
            id={`panel-${index}`}
            style={{
              maxHeight: openIndex === index ? '500px' : 0,
              overflow: 'hidden',
              transition: 'max-height 0.3s ease',
              padding: openIndex === index ? '0 16px 16px' : '0 16px',
            }}
          >
            {item.content}
          </div>
        </div>
      ))}
    </div>
  );
}
```

---

### Multi-Open Accordion

```jsx
function MultiAccordion({ items }) {
  const [openSet, setOpenSet] = useState(new Set());

  function toggle(id) {
    setOpenSet(prev => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });
  }

  return (
    <div>
      {items.map(item => (
        <AccordionItem
          key={item.id}
          item={item}
          isOpen={openSet.has(item.id)}
          onToggle={() => toggle(item.id)}
        />
      ))}
    </div>
  );
}
```

---

## 9. Modal and Dialog

---

### Requirements

- Open/close via prop or internal state
- Click outside to close
- Escape key to close
- Prevent background scroll
- Focus trap inside modal
- Portal rendering

---

### Implementation

```jsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, title, children }) {
  const overlayRef = useRef(null);

  // Close on Escape
  useEffect(() => {
    if (!isOpen) return;

    function handleKeyDown(e) {
      if (e.key === 'Escape') onClose();
    }

    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, [isOpen, onClose]);

  // Prevent background scroll
  useEffect(() => {
    if (isOpen) {
      document.body.style.overflow = 'hidden';
    }
    return () => {
      document.body.style.overflow = '';
    };
  }, [isOpen]);

  if (!isOpen) return null;

  return createPortal(
    <div
      ref={overlayRef}
      onClick={e => { if (e.target === overlayRef.current) onClose(); }}
      style={{
        position: 'fixed',
        inset: 0,
        background: 'rgba(0,0,0,0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 1000,
      }}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div
        style={{
          background: 'white',
          borderRadius: 8,
          padding: 24,
          maxWidth: '90vw',
          maxHeight: '90vh',
          overflow: 'auto',
          minWidth: 320,
        }}
      >
        <div style={{ display: 'flex', justifyContent: 'space-between', marginBottom: 16 }}>
          <h2 id="modal-title" style={{ margin: 0 }}>{title}</h2>
          <button onClick={onClose} aria-label="Close modal">✕</button>
        </div>
        {children}
      </div>
    </div>,
    document.body
  );
}
```

---

### Why createPortal

Portals render the modal at the end of `document.body`, outside the component's DOM hierarchy. This ensures:
- `z-index` works correctly (no parent stacking context interference)
- CSS `overflow: hidden` on parent containers does not clip the modal
- `position: fixed` behaves as expected

---

## 10. Toast and Notification System

---

### Requirements

- Multiple toasts can appear simultaneously
- Auto-dismiss after a timeout
- Manual dismiss
- Different types: success, error, warning, info
- Stack vertically

---

### Toast Context

```jsx
const ToastContext = createContext(null);

let toastId = 0;

export function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);

  function addToast({ message, type = 'info', duration = 3000 }) {
    const id = ++toastId;
    setToasts(prev => [...prev, { id, message, type }]);

    if (duration > 0) {
      setTimeout(() => removeToast(id), duration);
    }

    return id;
  }

  function removeToast(id) {
    setToasts(prev => prev.filter(t => t.id !== id));
  }

  return (
    <ToastContext.Provider value={{ addToast, removeToast }}>
      {children}
      <ToastContainer toasts={toasts} onRemove={removeToast} />
    </ToastContext.Provider>
  );
}

export function useToast() {
  const ctx = useContext(ToastContext);
  if (!ctx) throw new Error('useToast must be inside ToastProvider');
  return ctx;
}
```

---

### Toast Container

```jsx
function ToastContainer({ toasts, onRemove }) {
  return createPortal(
    <div
      style={{
        position: 'fixed',
        bottom: 24,
        right: 24,
        display: 'flex',
        flexDirection: 'column',
        gap: 8,
        zIndex: 9999,
      }}
    >
      {toasts.map(toast => (
        <Toast key={toast.id} toast={toast} onRemove={onRemove} />
      ))}
    </div>,
    document.body
  );
}

const typeStyles = {
  success: { background: '#4caf50', color: 'white' },
  error: { background: '#f44336', color: 'white' },
  warning: { background: '#ff9800', color: 'white' },
  info: { background: '#2196f3', color: 'white' },
};

function Toast({ toast, onRemove }) {
  return (
    <div
      style={{
        padding: '12px 16px',
        borderRadius: 4,
        display: 'flex',
        alignItems: 'center',
        gap: 12,
        minWidth: 240,
        ...typeStyles[toast.type],
      }}
      role="alert"
    >
      <span style={{ flex: 1 }}>{toast.message}</span>
      <button
        onClick={() => onRemove(toast.id)}
        style={{ background: 'none', border: 'none', color: 'inherit', cursor: 'pointer' }}
        aria-label="Dismiss notification"
      >
        ✕
      </button>
    </div>
  );
}
```

---

## 11. Form with Validation

---

### Requirements

- Validate on blur and on submit
- Show field-level error messages
- Disable submit while submitting
- Clear errors when user corrects them

---

### Form Hook Pattern

```jsx
function useForm(initialValues, validate) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  function handleChange(e) {
    const { name, value } = e.target;
    setValues(prev => ({ ...prev, [name]: value }));
    // Clear error as user types
    if (errors[name]) {
      setErrors(prev => ({ ...prev, [name]: '' }));
    }
  }

  function handleBlur(e) {
    const { name } = e.target;
    setTouched(prev => ({ ...prev, [name]: true }));
    const fieldErrors = validate(values);
    setErrors(prev => ({ ...prev, [name]: fieldErrors[name] || '' }));
  }

  async function handleSubmit(onSubmit) {
    return async (e) => {
      e.preventDefault();
      const validationErrors = validate(values);
      setErrors(validationErrors);
      setTouched(Object.keys(initialValues).reduce((acc, key) => ({ ...acc, [key]: true }), {}));

      if (Object.values(validationErrors).some(Boolean)) return;

      setIsSubmitting(true);
      try {
        await onSubmit(values);
      } finally {
        setIsSubmitting(false);
      }
    };
  }

  return { values, errors, touched, isSubmitting, handleChange, handleBlur, handleSubmit };
}
```

---

### Validation Function

```jsx
function validateLoginForm(values) {
  const errors = {};

  if (!values.email) {
    errors.email = 'Email is required';
  } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(values.email)) {
    errors.email = 'Invalid email address';
  }

  if (!values.password) {
    errors.password = 'Password is required';
  } else if (values.password.length < 8) {
    errors.password = 'Password must be at least 8 characters';
  }

  return errors;
}
```

---

## 12. File Upload with Preview

---

### Requirements

- Drag and drop or click to select
- Preview images before upload
- Show progress
- Validate file type and size
- Allow removing selected files

---

### Implementation

```jsx
function FileUpload({ accept = 'image/*', maxSizeMB = 5, onUpload }) {
  const [files, setFiles] = useState([]);
  const [dragOver, setDragOver] = useState(false);
  const inputRef = useRef(null);

  function validateFile(file) {
    if (file.size > maxSizeMB * 1024 * 1024) {
      return `File too large. Max ${maxSizeMB}MB.`;
    }
    if (accept !== '*' && !file.type.match(accept.replace('*', '.*'))) {
      return `Invalid file type. Accepted: ${accept}`;
    }
    return null;
  }

  function processFiles(fileList) {
    const newFiles = Array.from(fileList).map(file => {
      const error = validateFile(file);
      const preview = file.type.startsWith('image/')
        ? URL.createObjectURL(file)
        : null;
      return { file, preview, error, id: crypto.randomUUID() };
    });
    setFiles(prev => [...prev, ...newFiles]);
  }

  function removeFile(id) {
    setFiles(prev => {
      const removed = prev.find(f => f.id === id);
      if (removed?.preview) URL.revokeObjectURL(removed.preview); // cleanup
      return prev.filter(f => f.id !== id);
    });
  }

  function handleDrop(e) {
    e.preventDefault();
    setDragOver(false);
    processFiles(e.dataTransfer.files);
  }

  return (
    <div>
      <div
        onClick={() => inputRef.current?.click()}
        onDrop={handleDrop}
        onDragOver={e => { e.preventDefault(); setDragOver(true); }}
        onDragLeave={() => setDragOver(false)}
        style={{
          border: `2px dashed ${dragOver ? '#2196f3' : '#ccc'}`,
          padding: 32,
          textAlign: 'center',
          cursor: 'pointer',
          borderRadius: 8,
          background: dragOver ? '#e3f2fd' : 'transparent',
        }}
      >
        Drop files here or click to select
      </div>
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        multiple
        style={{ display: 'none' }}
        onChange={e => processFiles(e.target.files)}
      />
      <div style={{ display: 'flex', flexWrap: 'wrap', gap: 8, marginTop: 16 }}>
        {files.map(({ id, file, preview, error }) => (
          <div key={id} style={{ position: 'relative', width: 100 }}>
            {preview ? (
              <img src={preview} alt={file.name} style={{ width: '100%', height: 100, objectFit: 'cover' }} />
            ) : (
              <div style={{ width: 100, height: 100, background: '#f5f5f5', display: 'flex', alignItems: 'center', justifyContent: 'center' }}>
                {file.name.split('.').pop().toUpperCase()}
              </div>
            )}
            {error && <p style={{ color: 'red', fontSize: 11 }}>{error}</p>}
            <button
              onClick={() => removeFile(id)}
              style={{ position: 'absolute', top: 4, right: 4 }}
            >
              ✕
            </button>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

### Memory Management

`URL.createObjectURL` creates a reference to memory. Always call `URL.revokeObjectURL` when the preview is no longer needed to prevent memory leaks.

---

## 13. Real-Time Search Filter

---

### Requirements

- Filter a list as user types
- Case-insensitive matching
- Highlight matching text
- Show count of results

---

### Implementation

```jsx
function SearchFilter({ items }) {
  const [query, setQuery] = useState('');

  const filtered = useMemo(() => {
    if (!query.trim()) return items;
    const lower = query.toLowerCase();
    return items.filter(item =>
      item.name.toLowerCase().includes(lower) ||
      item.category.toLowerCase().includes(lower)
    );
  }, [items, query]);

  function highlight(text, query) {
    if (!query.trim()) return text;
    const parts = text.split(new RegExp(`(${query})`, 'gi'));
    return parts.map((part, i) =>
      part.toLowerCase() === query.toLowerCase() ? (
        <mark key={i} style={{ background: '#fff176', fontWeight: 600 }}>{part}</mark>
      ) : part
    );
  }

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
        aria-label="Search items"
      />
      <p>{filtered.length} result{filtered.length !== 1 ? 's' : ''}</p>
      <ul>
        {filtered.map(item => (
          <li key={item.id}>
            {highlight(item.name, query)} — {item.category}
          </li>
        ))}
      </ul>
      {filtered.length === 0 && <p>No results for "{query}"</p>}
    </div>
  );
}
```

---

## 14. Multi-Step Form and Wizard

---

### Requirements

- Multiple steps
- Step navigation (next/back)
- Validate before proceeding to next step
- Show progress indicator
- Collect data across all steps

---

### Implementation

```jsx
const STEPS = ['Personal Info', 'Contact', 'Review'];

function MultiStepForm() {
  const [step, setStep] = useState(0);
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    phone: '',
  });

  function updateField(field, value) {
    setFormData(prev => ({ ...prev, [field]: value }));
  }

  function validateStep(stepIndex) {
    switch (stepIndex) {
      case 0:
        return formData.name.trim().length >= 2;
      case 1:
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email);
      default:
        return true;
    }
  }

  function next() {
    if (validateStep(step)) {
      setStep(s => s + 1);
    }
  }

  function back() {
    setStep(s => s - 1);
  }

  async function submit() {
    await submitFormData(formData);
  }

  return (
    <div>
      {/* Progress indicator */}
      <div style={{ display: 'flex', marginBottom: 24 }}>
        {STEPS.map((label, index) => (
          <div
            key={label}
            style={{
              flex: 1,
              textAlign: 'center',
              padding: '8px 0',
              borderBottom: `3px solid ${index <= step ? '#2196f3' : '#e0e0e0'}`,
              color: index === step ? '#2196f3' : '#666',
              fontWeight: index === step ? 600 : 400,
            }}
          >
            {index + 1}. {label}
          </div>
        ))}
      </div>

      {/* Step content */}
      {step === 0 && (
        <input
          placeholder="Full Name"
          value={formData.name}
          onChange={e => updateField('name', e.target.value)}
        />
      )}
      {step === 1 && (
        <input
          type="email"
          placeholder="Email"
          value={formData.email}
          onChange={e => updateField('email', e.target.value)}
        />
      )}
      {step === 2 && (
        <div>
          <p>Name: {formData.name}</p>
          <p>Email: {formData.email}</p>
        </div>
      )}

      {/* Navigation */}
      <div style={{ display: 'flex', gap: 8, marginTop: 16 }}>
        {step > 0 && <button onClick={back}>Back</button>}
        {step < STEPS.length - 1 && (
          <button onClick={next} disabled={!validateStep(step)}>Next</button>
        )}
        {step === STEPS.length - 1 && (
          <button onClick={submit}>Submit</button>
        )}
      </div>
    </div>
  );
}
```

---

### State Design Decision

`formData` is a single object shared across all steps. This is the correct approach for a wizard — data accumulates across steps. The `step` index drives which form fields are visible.

---

## 15. Common Pitfalls in Machine Coding Rounds

---

### Not Handling Empty State

```jsx
// Missing — shows nothing when list is empty
return items.map(item => <Item key={item.id} item={item} />);

// Correct
if (items.length === 0) return <p>No items found.</p>;
return items.map(item => <Item key={item.id} item={item} />);
```

---

### Using Index as Key in Dynamic Lists

```jsx
// Wrong — breaks React reconciliation on filter/sort
{items.map((item, index) => <Item key={index} item={item} />)}

// Correct
{items.map(item => <Item key={item.id} item={item} />)}
```

---

### Forgetting to Handle Loading and Error States

```jsx
// Incomplete — no loading/error UX
const [data, setData] = useState([]);
useEffect(() => { fetchData().then(setData); }, []);

// Complete
const [data, setData] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  fetchData()
    .then(setData)
    .catch(err => setError(err.message))
    .finally(() => setLoading(false));
}, []);
```

---

### Race Conditions in Search

```jsx
// Wrong — old results can overwrite new ones
useEffect(() => {
  fetch(`/api/search?q=${query}`).then(r => r.json()).then(setResults);
}, [query]);

// Correct — cancel stale request
useEffect(() => {
  let cancelled = false;
  fetch(`/api/search?q=${query}`)
    .then(r => r.json())
    .then(data => { if (!cancelled) setResults(data); });
  return () => { cancelled = true; };
}, [query]);
```

---

## 16. Performance in Machine Coding

---

### When to Debounce

Debounce inputs that trigger expensive operations (API calls, large computations):
- Search inputs
- Filter inputs on large datasets
- Window resize handlers
- Autosave functionality

---

### When to Memoize

Use `useMemo` for:
- Filtering or sorting large lists
- Building derived data structures from props

Use `useCallback` for:
- Event handlers passed to child components that are wrapped in `React.memo`
- Functions listed as dependencies in `useEffect`

---

### When to Virtualize

Use virtualization when rendering lists of more than 100 items in a scrollable container. The default DOM can handle short lists efficiently.

---

## 17. Testing Your Implementation

Before calling the implementation done, manually test:

1. **Happy path** — normal usage works
2. **Empty state** — no data renders correctly
3. **Single item** — boundary case for list-based components
4. **Maximum values** — rating at max, list at end, form at last step
5. **Rapid interactions** — fast clicking, fast typing
6. **Keyboard navigation** — Tab, Enter, Escape, Arrow keys
7. **Error case** — what happens when the API fails?
8. **Accessibility** — can it be used without a mouse?

---

## 18. Code Organization Patterns

---

### Component File Structure

```text
ComponentName/
  index.jsx        — main component
  ComponentName.css — styles
  useComponentLogic.js — custom hook for complex logic
  helpers.js       — pure utility functions
  types.js         — TypeScript types (if using TS)
```

---

### Separating Logic from Presentation

```jsx
// Custom hook owns the logic
function useStarRating(maxRating, initialRating) {
  const [rating, setRating] = useState(initialRating);
  const [hovered, setHovered] = useState(0);

  return {
    rating,
    hovered,
    setRating,
    setHovered,
    displayed: hovered || rating,
  };
}

// Component owns only the presentation
function StarRating(props) {
  const { rating, hovered, setRating, setHovered, displayed } = useStarRating(
    props.maxRating,
    props.initialRating
  );

  return (/* JSX */);
}
```

---

## 19. Best Practices for Live Coding

---

### Communication

- Talk through your approach before writing code
- Explain your state decisions: "I'm keeping this in local state because only this component needs it"
- Call out trade-offs: "I'm not debouncing here, but I would in production"
- Ask before adding libraries: "Can I use a library, or should I implement this from scratch?"

---

### Incremental Building

```text
1. Build the simplest working version first
2. Add features one at a time
3. Handle edge cases after the core works
4. Refactor only if you have time
```

---

### What Interviewers Look For

| What they say | What they really check |
|---|---|
| "Build a search bar" | Do you debounce? Handle empty? Race conditions? |
| "Make this reusable" | Component API design, controlled vs uncontrolled |
| "What about performance?" | When to memoize, virtualize, debounce |
| "Handle edge cases" | Empty list, loading, error, boundary values |
| "Can you test this?" | Do you understand what to test and why? |

---

### Accessibility in Machine Coding

Interviewers at senior level note accessibility. Include at minimum:
- Semantic HTML (`<button>` not `<div onClick>`, `<nav>`, `<ul>` for lists)
- `aria-label` for icon-only buttons
- `role` and `aria-expanded` for accordions and dropdowns
- `aria-current="page"` for pagination active page
- Keyboard support for interactive widgets
