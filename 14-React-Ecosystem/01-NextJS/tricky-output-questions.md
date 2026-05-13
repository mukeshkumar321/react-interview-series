## Next.js — Tricky Output Questions

> These questions test deep understanding of Next.js App Router, data fetching, Server Actions, rendering strategies, and edge cases. Each question reflects real interview scenarios.

---

## 1. App Router vs Pages Router

### Q1

```jsx
// app/counter/page.tsx  (no "use client" directive)
import { useState } from 'react';

export default function CounterPage() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

#### ❓ What happens when this page is rendered?

<details>
<summary>✅ Answer</summary>

```txt
Error: You're importing a component that needs useState.
It only works in a Client Component but none of its parents are marked with "use client".
```

**Explanation:** In the App Router, every component is a Server Component by default. Server Components cannot use React hooks like `useState`. To fix this, add `'use client'` as the very first line of the file. The `'use client'` directive marks the file as the boundary where Client Component territory begins.

</details>

---

### Q2

```jsx
// app/dashboard/page.tsx
export default async function DashboardPage() {
  const data = await fetch('https://api.example.com/stats').then(r => r.json());
  return <h1>Visits: {data.visits}</h1>;
}
```

#### ❓ Where does this component run — server or browser?

<details>
<summary>✅ Answer</summary>

```txt
Server only. Zero JavaScript for this component is sent to the browser.
```

**Explanation:** There is no `'use client'` directive. The component is a Server Component. The `await fetch(...)` call happens entirely on the server during the request. The client receives the pre-rendered HTML `<h1>Visits: 42</h1>`. No JavaScript bundle for this component is included in the page.

</details>

---

### Q3

```jsx
// components/Modal.tsx
'use client';

import { useState } from 'react';
import ServerContent from './ServerContent'; // No "use client"

export default function Modal({ children }) {
  const [open, setOpen] = useState(false);
  return (
    <div>
      <button onClick={() => setOpen(o => !o)}>Toggle</button>
      {open && children}
    </div>
  );
}

// app/page.tsx (Server Component)
import Modal from '@/components/Modal';
import ServerContent from '@/components/ServerContent';

export default function Page() {
  return (
    <Modal>
      <ServerContent />
    </Modal>
  );
}
```

#### ❓ Is `ServerContent` treated as a Server Component or Client Component in this setup?

<details>
<summary>✅ Answer</summary>

```txt
ServerContent is a Server Component. The pattern is valid.
```

**Explanation:** `Modal` is a Client Component. `ServerContent` does not have `'use client'` so it remains a Server Component. The key is that `ServerContent` is imported and rendered in `app/page.tsx` (a Server Component) and passed as `children` — not imported inside `Modal.tsx`. Since `Modal.tsx` only receives `children` as a prop (not imports `ServerContent`), `ServerContent` is never pulled into the Client Component bundle. This is the "passing Server Components as children to Client Components" pattern.

</details>

---

### Q4

```jsx
// components/ThemeToggle.tsx
'use client';

import { useState } from 'react';
import HeavyAnalytics from './HeavyAnalytics'; // Has "use client" internally? No.

export default function ThemeToggle() {
  const [dark, setDark] = useState(false);
  return (
    <div>
      <button onClick={() => setDark(d => !d)}>Toggle</button>
      <HeavyAnalytics />
    </div>
  );
}
```

#### ❓ `HeavyAnalytics` has no `'use client'` directive. Is it a Server Component or Client Component when rendered in this tree?

<details>
<summary>✅ Answer</summary>

```txt
HeavyAnalytics becomes a Client Component.
```

**Explanation:** The `'use client'` directive creates a boundary. Everything in that file AND everything that file imports — all the way down the import chain — becomes part of the Client Component bundle. `HeavyAnalytics` is imported inside `ThemeToggle.tsx` which is a Client Component. Even without its own `'use client'`, it is treated as a Client Component because it is imported across the client boundary. Its JavaScript will be included in the browser bundle.

</details>

---

### Q5

```jsx
// pages/users/[id].tsx  (Pages Router)
export default function UserPage({ user }) {
  return <div>{user.name}</div>;
}

export async function getServerSideProps({ params }) {
  const user = await fetchUser(params.id);
  return { props: { user } };
}
```

```jsx
// app/users/[id]/page.tsx  (App Router)
export default async function UserPage({ params }) {
  const user = await fetchUser(params.id);
  return <div>{user.name}</div>;
}
```

#### ❓ What is the key behavioral difference between these two equivalent-looking pages?

<details>
<summary>✅ Answer</summary>

```txt
Pages Router: UserPage is a Client Component. getServerSideProps runs server-side and passes props.
App Router: The entire page.tsx component runs server-side. No props bridge is needed.
```

**Explanation:** In the Pages Router, all page components are Client Components by default — they hydrate in the browser. `getServerSideProps` runs on the server and the result is serialized and passed as props. In the App Router, the page component itself is a Server Component by default. The `async/await` happens directly in the component body on the server. There is no separate data-fetching function and no props serialization bridge.

</details>

---

## 2. Data Fetching

### Q6

```jsx
// app/products/page.tsx
export default async function ProductsPage() {
  const res = await fetch('https://api.example.com/products');
  const products = await res.json();
  return <ul>{products.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

#### ❓ How is this page rendered — statically or dynamically? What is the default caching behavior of this `fetch` call?

<details>
<summary>✅ Answer</summary>

```txt
Statically rendered (SSG equivalent).
Default caching: force-cache — the response is cached indefinitely until manually revalidated.
```

**Explanation:** In Next.js App Router (pre-Next.js 15), `fetch` defaults to `{ cache: 'force-cache' }`. The page is rendered at build time and the result is cached. The same HTML is served to all users without re-fetching from the API. To make it dynamic, explicitly pass `{ cache: 'no-store' }` or `{ next: { revalidate: N } }`. Note: In Next.js 15, the default changed to `no-store`, making this dynamic by default.

</details>

---

### Q7

```jsx
// app/posts/page.tsx
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 30 },
  });
  const posts = await res.json();
  return <PostList posts={posts} />;
}
```

#### ❓ A user visits this page 25 seconds after the last build. Another user visits 5 seconds later (35 seconds after build). What does each user see?

<details>
<summary>✅ Answer</summary>

```txt
User 1 (25s): Sees the cached (stale) page from build time.
              In the background, Next.js begins revalidating.
User 2 (35s): Sees the freshly generated page (or possibly still the old one
              if regeneration completed after User 1's request triggered it).
```

**Explanation:** This is ISR (Incremental Static Regeneration) with `revalidate: 30`. After 30 seconds, the cached page becomes "stale." The first request after the stale time triggers background regeneration — the requesting user still gets the stale cached page immediately. The next request after regeneration completes receives the fresh content. This "stale-while-revalidate" model ensures fast delivery while keeping content up-to-date.

</details>

---

### Q8

```jsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());
  return posts.map(post => ({ slug: post.slug }));
}

export default async function BlogPost({ params }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`).then(r => r.json());
  return <article>{post.content}</article>;
}
```

#### ❓ What does `generateStaticParams` do and what happens if a user visits `/blog/new-post` where `"new-post"` was not returned by `generateStaticParams`?

<details>
<summary>✅ Answer</summary>

```txt
generateStaticParams: Pre-builds all returned slugs at build time (SSG for each path).

/blog/new-post (not in params): By default, Next.js will attempt to render it
dynamically at request time (fallback behavior). It will be statically cached
after the first successful render.
```

**Explanation:** `generateStaticParams` is the App Router equivalent of `getStaticPaths`. It tells Next.js which dynamic path segments to pre-render at build time. For paths not included, the default fallback behavior in App Router is to render dynamically on first request and then cache the result. You can export `export const dynamicParams = false` to return a 404 for unknown paths instead.

</details>

---

### Q9

```jsx
// app/profile/page.tsx
async function getUserName() {
  const data = await fetch('https://api.example.com/me').then(r => r.json());
  return data.name;
}

async function getUserAvatar() {
  const data = await fetch('https://api.example.com/me').then(r => r.json());
  return data.avatar;
}

export default async function ProfilePage() {
  const name = await getUserName();
  const avatar = await getUserAvatar();
  return <div>{name} — {avatar}</div>;
}
```

#### ❓ How many HTTP requests are made to `https://api.example.com/me`?

<details>
<summary>✅ Answer</summary>

```txt
1 HTTP request — not 2.
```

**Explanation:** Next.js automatically deduplicates identical `fetch` calls within the same server render cycle. Both `getUserName` and `getUserAvatar` call the same URL with the same options. Next.js memoizes the first fetch result and returns it for the second call without making a new network request. This deduplication only applies within a single render — each new request gets fresh deduplication.

</details>

---

### Q10

```jsx
// app/dashboard/page.tsx
import { cookies } from 'next/headers';

export default async function Dashboard() {
  const cookieStore = cookies();
  const theme = cookieStore.get('theme')?.value ?? 'light';

  const data = await fetch('https://api.example.com/dashboard', {
    cache: 'force-cache',
  });

  return <div data-theme={theme}>{/* ... */}</div>;
}
```

#### ❓ Is this page statically or dynamically rendered? The `fetch` explicitly uses `force-cache`.

<details>
<summary>✅ Answer</summary>

```txt
Dynamically rendered on every request.
```

**Explanation:** Even though `fetch` uses `force-cache`, the page is dynamically rendered because it calls `cookies()` from `next/headers`. Any use of `cookies()`, `headers()`, or `searchParams` opts the entire component into dynamic rendering. Next.js determines rendering strategy at the component level — if any dynamic signal is detected, the entire page becomes dynamic. The `force-cache` on fetch still caches the fetch response, but the page itself regenerates on every request.

</details>

---

## 3. Server Actions

### Q11

```jsx
// app/actions.ts
'use server';

export async function submitForm(formData: FormData) {
  const name = formData.get('name');
  console.log('Server received:', name);
  // Save to database...
}

// app/contact/page.tsx  (Server Component)
import { submitForm } from '../actions';

export default function ContactPage() {
  return (
    <form action={submitForm}>
      <input name="name" placeholder="Your name" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

#### ❓ Where does `console.log('Server received:', name)` print? What happens if JavaScript is disabled in the browser?

<details>
<summary>✅ Answer</summary>

```txt
console.log prints in the server terminal/logs — NOT in the browser console.

With JavaScript disabled: The form still submits and the Server Action still runs.
The native HTML form submission mechanism handles it via a POST request.
```

**Explanation:** Server Actions run exclusively on the server. Their `console.log` output appears in the Node.js process terminal. Because the `<form action={serverAction}>` pattern uses native HTML form submission, it works without JavaScript — the browser sends a standard POST request and Next.js routes it to the Server Action. This is progressive enhancement by default.

</details>

---

### Q12

```jsx
// app/actions.ts
'use server';

export async function deleteItem(id: string) {
  await db.item.delete({ where: { id } });
  revalidatePath('/items');
}

// components/DeleteButton.tsx
'use client';

import { deleteItem } from '@/app/actions';

export default function DeleteButton({ id }: { id: string }) {
  return (
    <button onClick={() => deleteItem(id)}>Delete</button>
  );
}
```

#### ❓ Is this valid? Can a Client Component call a Server Action directly in an event handler?

<details>
<summary>✅ Answer</summary>

```txt
Yes — this is valid. A Client Component can import and call Server Actions.
```

**Explanation:** Server Actions can be called from Client Components in event handlers (not just form actions). When `deleteItem(id)` is called on button click, Next.js serializes the arguments, sends them to the server via an HTTP POST request, executes the action server-side, and returns the result. The `'use server'` file boundary ensures the function body never runs in the browser — only an RPC-like call is made.

</details>

---

### Q13

```jsx
// app/actions.ts
'use server';
import { revalidatePath, revalidateTag, redirect } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const post = await db.post.create({ data: { title } });
  
  revalidateTag('posts');
  redirect(`/posts/${post.id}`);
}
```

#### ❓ What does `revalidateTag('posts')` do and what is the effect of calling `redirect()` inside a Server Action?

<details>
<summary>✅ Answer</summary>

```txt
revalidateTag('posts'): Invalidates all fetch() cache entries tagged with 'posts'.
  Next time those routes are requested, they re-fetch data from the origin.

redirect('/posts/123'): Throws a special Next.js redirect error internally.
  The client is navigated to /posts/123 after the action completes.
  Do NOT wrap redirect() in a try/catch — the thrown value would be caught.
```

**Explanation:** `revalidateTag` provides surgical cache invalidation. Any `fetch` that used `{ next: { tags: ['posts'] } }` will be invalidated. `redirect()` in Next.js works by throwing an internal value that Next.js catches and uses to send a redirect response. If you wrap it in `try/catch`, you will accidentally swallow the redirect. Place `redirect()` outside try/catch blocks.

</details>

---

### Q14

```jsx
'use client';

import { useTransition } from 'react';
import { createTodo } from '@/app/actions';

export default function TodoForm() {
  const [isPending, startTransition] = useTransition();

  async function handleSubmit(formData: FormData) {
    startTransition(async () => {
      await createTodo(formData);
    });
  }

  return (
    <form action={handleSubmit}>
      <input name="title" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Saving...' : 'Add Todo'}
      </button>
    </form>
  );
}
```

#### ❓ What does `isPending` represent here and why is `useTransition` used with a Server Action?

<details>
<summary>✅ Answer</summary>

```txt
isPending: true while the Server Action is in flight (network request to server is pending).
           false after the action completes or fails.

useTransition is used to track the async state of the Server Action call so the UI
can show a loading state and disable the button during submission.
```

**Explanation:** Server Actions are inherently async — they involve a network round-trip to the server. `useTransition` wraps the action call, marking the transition as pending until it resolves. This provides the `isPending` boolean without needing a manual `useState` for loading. The button is disabled while pending, and the label switches to `'Saving...'`. This is the idiomatic pattern for Server Action loading states in Client Components.

</details>

---

### Q15

```jsx
// app/actions.ts
'use server';

export async function processData(callback: () => void, data: { items: Map<string, number> }) {
  // ...
}
```

#### ❓ What is wrong with this Server Action signature?

<details>
<summary>✅ Answer</summary>

```txt
Two problems:
1. callback: () => void — Functions cannot be passed as arguments to Server Actions.
2. Map<string, number> — Map is not a serializable plain JSON structure.

Server Action arguments must be serializable: strings, numbers, booleans,
arrays, plain objects, FormData, Date, URL, null, undefined.
```

**Explanation:** Server Actions receive arguments through a network boundary. Arguments are serialized (using React's serialization format) before being sent to the server. Functions are not serializable across this boundary — they would need to run on the client but the action runs on the server. `Map`, `Set`, class instances, and circular references are also not serializable in this context. Use plain arrays/objects instead.

</details>

---

## 4. Rendering

### Q16

```jsx
// app/page.tsx
export default function HomePage() {
  return <h1>Hello World</h1>;
}
```

```jsx
// app/dynamic/page.tsx
import { headers } from 'next/headers';

export default function DynamicPage() {
  const headersList = headers();
  const userAgent = headersList.get('user-agent');
  return <p>{userAgent}</p>;
}
```

#### ❓ Which page is statically rendered and which is dynamically rendered? Why?

<details>
<summary>✅ Answer</summary>

```txt
app/page.tsx: Statically rendered at build time (SSG).
app/dynamic/page.tsx: Dynamically rendered on every request (SSR equivalent).
```

**Explanation:** `HomePage` has no dynamic signals — no `cookies()`, `headers()`, `searchParams`, or `cache: 'no-store'` fetch calls. Next.js pre-renders it at build time and serves the static HTML for every request. `DynamicPage` calls `headers()` from `next/headers`, which can only be resolved at request time (different users send different headers). This dynamic signal forces the page to render per-request.

</details>

---

### Q17

```jsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import FastStats from './FastStats';     // resolves in 50ms
import SlowChart from './SlowChart';     // resolves in 2000ms

export default function Dashboard() {
  return (
    <main>
      <FastStats />
      <Suspense fallback={<div>Loading chart...</div>}>
        <SlowChart />
      </Suspense>
    </main>
  );
}
```

#### ❓ In what order does the browser receive and display content?

<details>
<summary>✅ Answer</summary>

```txt
1. ~50ms: Browser receives the HTML shell including <FastStats /> rendered content
          and the <div>Loading chart...</div> fallback placeholder.
2. ~2000ms: The <SlowChart /> content streams in and replaces the fallback.
```

**Explanation:** Next.js App Router uses React's streaming with Suspense. The server starts sending the HTTP response immediately — it does not wait for all components to resolve. `FastStats` resolves quickly and its HTML is flushed to the browser. `SlowChart` is still fetching, so the `fallback` is sent in its place. When `SlowChart` resolves, its HTML streams down and React swaps it into the DOM client-side. The page becomes useful in ~50ms instead of ~2000ms.

</details>

---

### Q18

```text
app/
  layout.tsx
  loading.tsx      ← content: <div>Loading page...</div>
  page.tsx         ← slow async component (takes 3 seconds)
  about/
    page.tsx       ← fast component
```

#### ❓ What does `loading.tsx` at the root level do and when does the user see its content?

<details>
<summary>✅ Answer</summary>

```txt
loading.tsx automatically wraps page.tsx in a <Suspense> boundary.
The user sees <div>Loading page...</div> immediately while page.tsx is resolving.
After 3 seconds, page.tsx replaces the loading UI.

The about/page.tsx does NOT use this loading.tsx (it has no loading.tsx in its own segment).
```

**Explanation:** `loading.tsx` is a special Next.js file that creates an automatic `<Suspense>` boundary around `page.tsx` in the same directory. Next.js streams the layout and the loading fallback immediately, then streams the page content when it resolves. This is a declarative way to add loading states without manually wrapping in `<Suspense>`. Each route segment can have its own `loading.tsx`.

</details>

---

### Q19

```jsx
// app/product/[id]/page.tsx
import { Suspense } from 'react';
import ProductInfo from './ProductInfo';       // Static — same for all
import PersonalizedRecs from './PersonalizedRecs';  // Dynamic — per user

export default function ProductPage() {
  return (
    <>
      <ProductInfo />
      <Suspense fallback={<RecsLoading />}>
        <PersonalizedRecs />
      </Suspense>
    </>
  );
}
```

#### ❓ This is an example of what Next.js rendering strategy? What is the key benefit?

<details>
<summary>✅ Answer</summary>

```txt
This is Partial Prerendering (PPR) — available experimentally in Next.js 14.

Key benefit: The static shell (<ProductInfo />) is served from CDN instantly.
             The dynamic part (<PersonalizedRecs />) streams in at request time.
             No full-page SSR penalty. No full-page static limitation.
```

**Explanation:** PPR combines static and dynamic rendering on a single page. At build time, Next.js generates the static shell. At request time, dynamic `<Suspense>` boundaries are filled via streaming. Users see the product info immediately (from CDN) while personalized recommendations load. This is the best of both SSG (speed) and SSR (freshness) on the same page.

</details>

---

### Q20

```jsx
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <nav>Navigation</nav>
        {children}
      </body>
    </html>
  );
}
```

A user navigates from `/about` to `/contact`. Which of the following is true?

A) `RootLayout` unmounts and remounts  
B) `RootLayout` stays mounted, only `children` updates  
C) The entire page does a full browser refresh  

#### ❓ What is the answer and why?

<details>
<summary>✅ Answer</summary>

```txt
B) RootLayout stays mounted, only children updates.
```

**Explanation:** Layouts in App Router are **persistent** — they do not unmount on navigation between pages within their segment. The `<nav>Navigation</nav>` remains in the DOM without re-rendering. Only the `{children}` (the page component) updates. This is a critical advantage of the App Router layout system over Pages Router's `_app.tsx` — scroll position, focus, and layout state are preserved during navigation. Only `template.tsx` files remount on navigation.

</details>

---

## 5. Edge Cases

### Q21

```jsx
// app/notifications/page.tsx  (Server Component — no "use client")
import { useEffect } from 'react';

export default function NotificationsPage() {
  useEffect(() => {
    const ws = new WebSocket('wss://api.example.com/ws');
    ws.onmessage = (e) => console.log(e.data);
    return () => ws.close();
  }, []);

  return <div>Notifications</div>;
}
```

#### ❓ What happens when Next.js tries to render this page?

<details>
<summary>✅ Answer</summary>

```txt
Build/runtime error:
"Error: You're importing a component that needs useEffect.
It only works in a Client Component but none of its parents are marked with 'use client'."
```

**Explanation:** `useEffect` is a hook that runs after the component mounts in the browser. Server Components have no lifecycle — they run once on the server and never in the browser. Using any React hook (`useEffect`, `useState`, `useRef`, `useContext`, `useCallback`, `useMemo`) in a Server Component is an error. The fix is to add `'use client'` to the top of the file.

</details>

---

### Q22

```jsx
// app/timer/page.tsx  (Server Component — no "use client")
export default function TimerPage() {
  const now = new Date().toISOString();
  return <p>Page loaded at: {now}</p>;
}
```

#### ❓ Every user who visits this page sees "Page loaded at: 2024-01-15T10:00:00.000Z" (the build time). How would you make it show the actual current time for each user?

<details>
<summary>✅ Answer</summary>

```txt
Option 1: Make the page dynamic by adding a dynamic signal.
  import { cookies } from 'next/headers'; // forces dynamic rendering
  Or: export const dynamic = 'force-dynamic';

Option 2: Move the clock to a Client Component.
  'use client';
  const [now, setNow] = useState(new Date().toISOString());
```

**Explanation:** The Server Component runs at build time (statically rendered). `new Date()` captures the build time, not the request time. To get the current time per request, you can: (a) force dynamic rendering so the server component runs on each request, or (b) move the timestamp to a Client Component where it runs in the browser. For a live clock, `useEffect` + `setInterval` in a Client Component is the correct approach.

</details>

---

### Q23

```jsx
// app/layout.tsx
export const metadata = {
  title: 'My App',
  description: 'The best app',
};

// app/about/page.tsx
export const metadata = {
  title: 'About Us',
};

export default function AboutPage() {
  return <h1>About</h1>;
}
```

#### ❓ What `<title>` does the browser show when visiting `/about`?

<details>
<summary>✅ Answer</summary>

```txt
<title>About Us</title>
```

**Explanation:** Next.js merges metadata from the layout hierarchy. When a page exports `metadata`, it overrides the corresponding fields from parent layouts. The `about/page.tsx` exports `title: 'About Us'`, which overrides `title: 'My App'` from the root layout. The `description: 'The best app'` from the layout is inherited since `about/page.tsx` does not override it. For dynamic titles, use `generateMetadata()` instead of the static `metadata` export.

</details>

---

### Q24

```jsx
// components/HeroImage.tsx
import Image from 'next/image';
import heroImg from '/public/hero.jpg';

export default function HeroImage() {
  return (
    <Image
      src={heroImg}
      alt="Hero banner"
      width={1200}
      height={600}
    />
  );
}

// vs.

export default function HeroImagePlain() {
  return (
    <img src="/hero.jpg" alt="Hero banner" width={1200} height={600} />
  );
}
```

#### ❓ List three concrete differences between using `<Image>` from `next/image` versus a plain `<img>` tag.

<details>
<summary>✅ Answer</summary>

```txt
1. Format conversion: <Image> automatically serves WebP or AVIF based on browser support.
   <img> serves the original format (JPEG/PNG).

2. Lazy loading: <Image> lazy loads by default — only fetches when near the viewport.
   <img> loads eagerly unless you manually add loading="lazy".

3. CLS prevention: <Image> reserves the exact space before the image loads
   (uses aspect ratio from width/height), preventing layout shift.
   <img> without explicit dimensions causes layout shift as the image loads.

Bonus: <Image> serves images through Next.js's optimization API (/_next/image)
which applies compression, resizing to exact display size, and CDN caching.
```

**Explanation:** The `next/image` component is not just a convenience wrapper — it provides automatic format selection, lazy loading, layout-shift prevention, and server-side resizing through the `/_next/image` URL endpoint. Use `priority` prop on above-the-fold LCP images to preload them eagerly.

</details>

---

### Q25

```jsx
// app/admin/page.tsx
import { cookies, headers } from 'next/headers';

export default async function AdminPage() {
  // Reading cookies and headers
  const cookieStore = cookies();
  const authToken = cookieStore.get('auth-token')?.value;
  
  const headersList = headers();
  const userAgent = headersList.get('user-agent');
  
  if (!authToken) {
    return <p>Unauthorized</p>;
  }
  
  const data = await fetch('https://api.internal.com/admin', {
    next: { revalidate: 60 },  // ISR: revalidate every 60 seconds
    headers: { Authorization: `Bearer ${authToken}` },
  });
  
  return <div>{/* ... */}</div>;
}
```

#### ❓ The developer wants ISR behavior with `revalidate: 60`. Will this work as expected?

<details>
<summary>✅ Answer</summary>

```txt
No — ISR will NOT work. The page will be dynamically rendered on every request.

The revalidate: 60 on the fetch() call will still cache the fetch response,
but the page itself renders on every request because cookies() and headers()
are dynamic APIs.
```

**Explanation:** `revalidate: 60` on a `fetch()` caches the network response for 60 seconds. But ISR applies at the **page** level. The page uses `cookies()` and `headers()` which are request-time signals — they force the entire page into dynamic rendering mode. The page will SSR on every request, bypassing static/ISR generation. The fetch response may be cached, but the component re-runs for each request. For ISR with auth, you typically extract the auth check into middleware and keep the page component free of dynamic APIs.

</details>

---

## Topics Covered

| Category | Questions | Concepts Tested |
|---|---|---|
| App Router vs Pages Router | Q1–Q5 | Server vs Client Components, 'use client' boundary, component rendering location, passing SC as children |
| Data Fetching | Q6–Q10 | Default fetch caching, ISR revalidate, generateStaticParams, fetch deduplication, dynamic rendering triggers |
| Server Actions | Q11–Q15 | Form submission, progressive enhancement, calling from Client Component, revalidatePath, useTransition, serialization rules |
| Rendering | Q16–Q20 | Static vs dynamic rendering, Streaming with Suspense, loading.tsx, PPR, layout persistence |
| Edge Cases | Q21–Q25 | useEffect in Server Component, build-time vs request-time, Metadata API, Image optimization benefits, cookies causing dynamic rendering |
