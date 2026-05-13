# Next.js

## Table of Contents

1. [What is Next.js](#1-what-is-nextjs)
2. [App Router vs Pages Router](#2-app-router-vs-pages-router)
3. [Rendering Strategies](#3-rendering-strategies)
4. [App Router File Structure](#4-app-router-file-structure)
5. [Server Components vs Client Components](#5-server-components-vs-client-components)
6. [Data Fetching in App Router](#6-data-fetching-in-app-router)
7. [Server Actions](#7-server-actions)
8. [Pages Router Data Fetching](#8-pages-router-data-fetching)
9. [File-Based Routing](#9-file-based-routing)
10. [Dynamic Routes](#10-dynamic-routes)
11. [Layout System](#11-layout-system)
12. [Middleware](#12-middleware)
13. [Image Optimization](#13-image-optimization)
14. [API Routes](#14-api-routes)
15. [Next.js Performance Features](#15-nextjs-performance-features)
16. [Deployment](#16-deployment)
17. [Common Interview Questions](#17-common-interview-questions)
18. [Common Mistakes](#18-common-mistakes)
19. [Best Practices](#19-best-practices)

---

## 1. What is Next.js

Next.js is a **production-grade React framework** built and maintained by Vercel. It extends React with capabilities that are needed in real-world applications but are not provided by React itself — routing, server-side rendering, static generation, API routes, image optimization, and more.

React is a UI library. Next.js is the framework that tells React *where to run*, *when to render*, and *how to serve* the application.

### Core Capabilities

- **File-based routing** — the file system defines URL structure, no router configuration needed
- **Multiple rendering strategies** — CSR, SSR, SSG, ISR, PPR — per page or even per component
- **Built-in API layer** — create backend endpoints alongside frontend code
- **Server Components** — run React components on the server with zero JS shipped to the client
- **Server Actions** — call server-side functions from forms and event handlers without a separate API
- **Optimized primitives** — `next/image`, `next/font`, `next/script` with built-in performance best practices

### Why Next.js Over Plain React

| Problem with plain React | Next.js solution |
|---|---|
| No routing built-in | File-based routing |
| SEO poor (empty HTML shell) | SSR / SSG pre-renders full HTML |
| Manual code splitting | Automatic per-route splitting |
| No API server | Built-in API routes |
| Image optimization manual | `next/image` with WebP, lazy load, CDN |
| Font layout shift | `next/font` with inline CSS variables |

### Vercel Ecosystem

Next.js is open-source (MIT) but Vercel is the primary maintainer. Deploying to Vercel provides zero-configuration support for all Next.js features (ISR, Edge Middleware, Streaming, etc.). Self-hosting on Node.js is fully supported but some features (like Edge runtime) require additional setup.

---

## 2. App Router vs Pages Router

Next.js 13 introduced the **App Router** — a complete rethink of routing, layouts, and data fetching, built on top of React Server Components. As of Next.js 14/15, the App Router is the **recommended default**.

### Pages Router (Legacy, Still Supported)

- Directory: `/pages`
- All components are Client Components by default (hydrated in browser)
- Data fetching via special exported functions: `getServerSideProps`, `getStaticProps`, `getStaticPaths`
- Layouts via `_app.tsx` and manual wrapping
- API routes in `/pages/api`

```jsx
// pages/users/[id].tsx — Pages Router
export default function UserPage({ user }) {
  return <div>{user.name}</div>;
}

export async function getServerSideProps({ params }) {
  const user = await fetchUser(params.id);
  return { props: { user } };
}
```

### App Router (Current, Recommended)

- Directory: `/app`
- All components are **Server Components by default**
- Data fetching directly in components using `async/await`
- Nested layouts via `layout.tsx` files
- API routes via `route.ts` files

```jsx
// app/users/[id]/page.tsx — App Router
export default async function UserPage({ params }) {
  const user = await fetchUser(params.id); // runs on server
  return <div>{user.name}</div>;
}
```

### Comparison Table

| Feature | Pages Router | App Router |
|---|---|---|
| Directory | `/pages` | `/app` |
| Default component type | Client Component | Server Component |
| Data fetching | `getServerSideProps` etc. | `async/await` in component |
| Layouts | `_app.tsx` (global only) | Nested `layout.tsx` per segment |
| Streaming | Not supported | Built-in with `<Suspense>` |
| Server Actions | Not supported | Supported |
| Caching | ISR via `revalidate` | Fine-grained `fetch` cache options |
| React Server Components | No | Yes |
| Maturity | Stable, battle-tested | Stable as of Next.js 14 |

### When to Use Which

**Use App Router** for all new projects. It is the default and gives access to RSC, Server Actions, nested layouts, and Streaming.

**Use Pages Router** if maintaining a legacy Next.js project or if a critical third-party library is incompatible with RSC.

Both routers can **coexist** in the same project during migration. A page in `/pages` and a page in `/app` at different paths will both work simultaneously.

---

## 3. Rendering Strategies

Understanding *when* HTML is generated and *where* data fetching happens is fundamental to Next.js architecture decisions.

### CSR — Client-Side Rendering

HTML is empty on the first load. JavaScript loads, runs in the browser, fetches data, and renders the UI.

```text
Browser requests /page
     ↓
Server sends empty HTML + JS bundle
     ↓
Browser executes JS
     ↓
JS fetches data (useEffect / React Query)
     ↓
UI renders
```

**When to use:** Highly interactive dashboards, admin panels, pages behind auth that do not need SEO.

**In Next.js App Router:** Use `"use client"` + `useEffect` or React Query.

### SSR — Server-Side Rendering

HTML is generated **on every request** on the server. The client receives fully-rendered HTML.

```text
Browser requests /page
     ↓
Server fetches data + renders HTML
     ↓
Browser receives full HTML (immediately visible)
     ↓
React hydrates (attaches event listeners)
     ↓
Page is interactive
```

**In App Router:** Any Server Component that uses `fetch` with `{ cache: 'no-store' }` or reads cookies/headers is dynamically rendered — equivalent to SSR.

**In Pages Router:** `getServerSideProps`

### SSG — Static Site Generation

HTML is generated **at build time** (`next build`). The same HTML is served to every user.

```text
next build runs
     ↓
Server fetches data + renders HTML → saves to .html file
     ↓
Browser requests /page
     ↓
CDN returns pre-built HTML (fast, no server work per request)
     ↓
React hydrates
```

**When to use:** Marketing pages, blog posts, documentation — content that does not change often.

**In App Router:** Default behavior for Server Components using `fetch` with default caching (`force-cache`).

**In Pages Router:** `getStaticProps`

### ISR — Incremental Static Regeneration

Combines SSG (fast delivery) with automatic background revalidation. Pages are pre-built but refreshed after a set interval.

```text
First request after build: serve existing static HTML
     ↓
In background: regenerate page with fresh data
     ↓
Subsequent requests: serve the newly generated HTML
```

**In App Router:**
```jsx
// app/posts/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 }, // regenerate at most once every 60 seconds
  });
  return res.json();
}
```

**In Pages Router:**
```jsx
export async function getStaticProps() {
  const posts = await getPosts();
  return { props: { posts }, revalidate: 60 };
}
```

### PPR — Partial Prerendering (Next.js 14 Experimental)

A single page can have a **static shell** (prerendered at build time) with **dynamic holes** (filled at request time via Streaming). No full-page SSR penalty; no full-page static limitation.

```text
Build time: generate static HTML shell
     ↓
Request time: stream dynamic content into shell holes
     ↓
User sees static shell instantly, dynamic parts arrive later
```

```jsx
// app/product/[id]/page.tsx
import { Suspense } from 'react';
import StaticHeader from './StaticHeader';
import DynamicRecommendations from './DynamicRecommendations';

export default function ProductPage() {
  return (
    <>
      <StaticHeader />  {/* prerendered at build */}
      <Suspense fallback={<Skeleton />}>
        <DynamicRecommendations />  {/* streamed at request time */}
      </Suspense>
    </>
  );
}
```

### Rendering Strategy Decision Chart

```text
Is the content user-specific or must be real-time?
          ↓ yes                       ↓ no
    SSR (no-store)             Does it change at all?
                                  ↓ yes         ↓ no
                              ISR / PPR          SSG
```

---

## 4. App Router File Structure

The `/app` directory uses a **convention-based file system** where specific filenames carry special meaning to the framework.

### Special Files

| Filename | Purpose |
|---|---|
| `page.tsx` | Defines the UI for a route segment (makes URL publicly accessible) |
| `layout.tsx` | Wraps children and persists across navigations in the segment |
| `loading.tsx` | Automatic Suspense boundary fallback for the segment |
| `error.tsx` | Error boundary for the segment |
| `not-found.tsx` | Rendered when `notFound()` is called |
| `route.ts` | API route handler (GET, POST, PUT, DELETE, etc.) |
| `template.tsx` | Like layout but re-mounts on every navigation |
| `default.tsx` | Fallback UI for parallel routes |

### Example Directory Structure

```text
app/
  layout.tsx            ← Root layout (required, must contain <html><body>)
  page.tsx              ← / route
  loading.tsx           ← Root loading UI
  error.tsx             ← Root error boundary
  globals.css
  about/
    page.tsx            ← /about route
  users/
    layout.tsx          ← Persistent layout for /users and /users/*
    page.tsx            ← /users route
    [id]/
      page.tsx          ← /users/:id route
      loading.tsx       ← Loading UI for individual user
      error.tsx         ← Error boundary for individual user
  api/
    users/
      route.ts          ← GET/POST /api/users
  (auth)/               ← Route group — no URL segment added
    login/
      page.tsx          ← /login route
    signup/
      page.tsx          ← /signup route
```

### Root Layout (Required)

Every App Router project must have an `app/layout.tsx` that includes `<html>` and `<body>`:

```jsx
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>{children}</body>
    </html>
  );
}
```

### page.tsx vs route.ts

- `page.tsx` — exports a React component, makes the URL accessible to browsers
- `route.ts` — exports HTTP handler functions, creates an API endpoint

A route segment **cannot have both** `page.tsx` and `route.ts`.

---

## 5. Server Components vs Client Components

This is the most architecturally significant concept in the App Router. Understanding the trade-offs is essential for interviews.

### Server Components (Default)

Server Components run **exclusively on the server** during the request. Their rendered output (HTML) is sent to the client. **No JavaScript** for Server Components is shipped to the browser.

**Capabilities:**
- Direct access to server resources (database, file system, environment variables)
- `async/await` at the component level
- Reduced client bundle size

**Limitations:**
- Cannot use `useState`, `useReducer`, `useEffect`, or any hook
- Cannot use browser APIs (`window`, `document`, `localStorage`)
- Cannot attach event handlers (`onClick`, `onChange`)

```jsx
// app/users/page.tsx — Server Component (no "use client")
import { db } from '@/lib/db';

export default async function UsersPage() {
  // runs on server — db credentials never exposed to client
  const users = await db.user.findMany();

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}
```

### Client Components

Client Components are marked with `"use client"` at the top of the file. They are pre-rendered on the server (for initial HTML) **and** hydrated and executed in the browser.

```jsx
'use client';

import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### Comparison Table

| Capability | Server Component | Client Component |
|---|---|---|
| `useState` / `useReducer` | ❌ | ✅ |
| `useEffect` | ❌ | ✅ |
| `onClick`, `onChange` | ❌ | ✅ |
| `async/await` in component body | ✅ | ❌ (use hooks instead) |
| Access DB / server secrets | ✅ | ❌ |
| `fetch` at component level | ✅ | ❌ (use hook) |
| Browser APIs | ❌ | ✅ |
| Contributes to JS bundle | ❌ (none) | ✅ |
| Rendered on server | ✅ | ✅ (initial render) + browser |

### "use client" Boundary

`"use client"` does not mean "only runs on client". It means: **"this is the boundary between server and client code."** Everything in this file and everything it imports becomes a Client Component.

```text
Server Component tree
         ↓
   "use client" boundary
         ↓
   Client Component subtree
   (all imports become client code too)
```

### Composition Pattern

Server Components **can** render Client Components. Client Components **cannot** directly render Server Components — but can receive them as `children`.

```jsx
// Server Component — can import and render Client Component
import InteractiveWidget from './InteractiveWidget'; // "use client"

export default async function Page() {
  const data = await fetchData();
  return (
    <div>
      <h1>{data.title}</h1>
      <InteractiveWidget />  {/* Client Component rendered inside Server Component */}
    </div>
  );
}
```

```jsx
// Passing Server Component as children to a Client Component
// app/page.tsx (Server Component)
import Modal from './Modal';               // "use client"
import ServerContent from './ServerContent'; // Server Component

export default function Page() {
  return (
    <Modal>
      <ServerContent />  {/* Server Component passed as children — valid */}
    </Modal>
  );
}
```

### When to Use Each

**Use Server Component when:**
- Fetching data from the database or an API
- Accessing backend resources or environment secrets
- Rendering large static content
- Keeping sensitive logic off the client bundle

**Use Client Component when:**
- Using `useState`, `useEffect`, or other hooks
- Handling user interaction events
- Using browser APIs (`window`, `navigator`, `localStorage`)
- Integrating third-party libraries that use browser APIs

---

## 6. Data Fetching in App Router

The App Router data fetching model is a major shift from Pages Router. Data fetching happens directly in Server Components using native `fetch` with extended cache options.

### fetch() in Server Components

```jsx
// app/posts/page.tsx
export default async function PostsPage() {
  const res = await fetch('https://api.example.com/posts');
  // fetch is automatically deduped — same URL called in same render only fetches once
  const posts = await res.json();

  return <PostList posts={posts} />;
}
```

### Cache Control Options

Next.js extends the native `fetch` API with a `next` option:

```jsx
// Static — cached indefinitely (SSG equivalent, this is the default)
fetch(url, { cache: 'force-cache' });

// Dynamic — no cache, fresh on every request (SSR equivalent)
fetch(url, { cache: 'no-store' });

// ISR — revalidate after N seconds
fetch(url, { next: { revalidate: 60 } });

// Tag-based revalidation — precise cache invalidation
fetch(url, { next: { tags: ['posts'] } });
```

### Request Memoization

Within a single server render, Next.js **automatically deduplicates** identical `fetch` calls. You can call `fetch(url)` in multiple components in the same render tree and only one HTTP request is made.

```jsx
// Both components call the same URL — only one HTTP request is made
async function Title() {
  const data = await fetch('/api/post/1').then(r => r.json()); // fetch 1
  return <h1>{data.title}</h1>;
}

async function Author() {
  const data = await fetch('/api/post/1').then(r => r.json()); // deduplicated
  return <span>{data.author}</span>;
}
```

### Dynamic vs Static Rendering

A Server Component is **statically rendered** (SSG) unless it opts into dynamic rendering. Dynamic rendering happens when:

- `fetch` uses `{ cache: 'no-store' }`
- The component reads `cookies()` or `headers()` from `next/headers`
- The component accesses the `searchParams` prop
- `noStore()` from `next/cache` is called explicitly

```jsx
import { cookies } from 'next/headers';

// This component is now dynamically rendered on every request
export default async function Dashboard() {
  const cookieStore = cookies();
  const token = cookieStore.get('auth-token');
  const data = await fetch('/api/dashboard', {
    headers: { Authorization: `Bearer ${token?.value}` },
    cache: 'no-store',
  });
  return <DashboardUI data={await data.json()} />;
}
```

### Parallel Data Fetching

Avoid sequential `await` calls when requests are independent — use `Promise.all`:

```jsx
// Sequential — total time = A + B (slow)
async function Page() {
  const user = await fetchUser();    // 500ms
  const posts = await fetchPosts();  // 300ms — waits for user unnecessarily
  // total: ~800ms
}

// Parallel — total time = max(A, B) (fast)
async function Page() {
  const [user, posts] = await Promise.all([
    fetchUser(),    // 500ms
    fetchPosts(),   // 300ms — starts immediately
  ]);
  // total: ~500ms
}
```

### Streaming with Suspense

Wrap slow data-fetching components in `<Suspense>` to stream them independently:

```jsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import UserProfile from './UserProfile';  // fast
import Analytics from './Analytics';     // slow

export default function Dashboard() {
  return (
    <main>
      <UserProfile />  {/* renders immediately */}
      <Suspense fallback={<AnalyticsSkeleton />}>
        <Analytics />  {/* streams in when ready, does not block UserProfile */}
      </Suspense>
    </main>
  );
}
```

---

## 7. Server Actions

Server Actions allow you to define **async server-side functions** that can be called directly from client-side code — forms, event handlers, or anywhere in React. They eliminate the need for a separate API route for mutations.

### Defining a Server Action

```jsx
// app/actions.ts
'use server';  // marks entire module — all exports are Server Actions

import { revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  await db.post.create({ data: { title } });
  revalidatePath('/posts');  // invalidate the cached posts page
}
```

### Using in a Form (Server Component)

```jsx
// app/posts/new/page.tsx (Server Component)
import { createPost } from '../actions';

export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Post title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

This form works **even without JavaScript enabled** because it uses native HTML form submission routed through Next.js's server infrastructure.

### Using in Client Components

```jsx
'use client';
import { createPost } from '@/app/actions';

export default function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

### With useTransition for Pending State

```jsx
'use client';
import { useTransition } from 'react';
import { createPost } from '@/app/actions';

export default function NewPostForm() {
  const [isPending, startTransition] = useTransition();

  function handleSubmit(formData: FormData) {
    startTransition(async () => {
      await createPost(formData);
    });
  }

  return (
    <form action={handleSubmit}>
      <input name="title" />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>
    </form>
  );
}
```

### Server Action Rules

- Must be `async` functions
- Must be in files marked with `'use server'`, or inline in Server Components with `'use server'` inside the function body
- Can return serializable values or `void`
- Can call `revalidatePath`, `revalidateTag`, `redirect`, `cookies`, `headers`
- Arguments and return values must be serializable (no class instances, no functions)

---

## 8. Pages Router Data Fetching

The Pages Router uses special **exported async functions** that Next.js calls at the right time during the build or request lifecycle.

### getServerSideProps

Runs on the **server on every request**. Equivalent to SSR. The returned props are passed to the page component.

```jsx
// pages/users/[id].tsx
export default function UserPage({ user }) {
  return <div>{user.name}</div>;
}

export async function getServerSideProps(context) {
  const { params, req, res, query } = context;
  const user = await fetchUser(params.id);

  if (!user) {
    return { notFound: true }; // renders 404 page
  }

  return {
    props: { user },
  };
}
```

### getStaticProps

Runs at **build time**. Returns props baked into the static HTML.

```jsx
// pages/posts/index.tsx
export default function PostsPage({ posts }) {
  return <PostList posts={posts} />;
}

export async function getStaticProps() {
  const posts = await fetchPosts();
  return {
    props: { posts },
    revalidate: 60,  // ISR: regenerate in background after 60 seconds
  };
}
```

### getStaticPaths

Required when using `getStaticProps` with **dynamic routes**. Tells Next.js which paths to pre-build at build time.

```jsx
// pages/posts/[slug].tsx
export async function getStaticPaths() {
  const posts = await fetchPosts();
  return {
    paths: posts.map(p => ({ params: { slug: p.slug } })),
    fallback: false,         // 404 for any unknown path
    // fallback: 'blocking'  — SSR on first request for unknown path, then cache as static
    // fallback: true        — show loading state, then fetch in background
  };
}

export async function getStaticProps({ params }) {
  const post = await fetchPost(params.slug);
  return { props: { post } };
}
```

### Pages Router Data Fetching Summary

| Function | When it runs | Use case |
|---|---|---|
| `getServerSideProps` | Every request (server) | User-specific data, real-time data |
| `getStaticProps` | Build time | Public content, blog posts |
| `getStaticPaths` | Build time (with dynamic routes) | Define which dynamic paths to pre-build |
| `revalidate` in `getStaticProps` | Background after interval | ISR |

---

## 9. File-Based Routing

Next.js derives URL structure directly from the file system. No route configuration files are needed.

### App Router Routing Rules

```text
File path                                  →  URL
─────────────────────────────────────────────────────
app/page.tsx                               →  /
app/about/page.tsx                         →  /about
app/blog/page.tsx                          →  /blog
app/blog/[slug]/page.tsx                   →  /blog/:slug
app/users/[id]/settings/page.tsx           →  /users/:id/settings
app/(marketing)/about/page.tsx             →  /about  (group ignored in URL)
app/(auth)/login/page.tsx                  →  /login  (group ignored in URL)
```

### Accessing Route Parameters

```jsx
// app/users/[id]/page.tsx
export default function UserPage({ params }: { params: { id: string } }) {
  return <div>User: {params.id}</div>;
}
```

### Accessing Search Parameters

```jsx
// app/search/page.tsx
export default function SearchPage({
  searchParams,
}: {
  searchParams: { q?: string; page?: string };
}) {
  const query = searchParams.q ?? '';
  const page = Number(searchParams.page ?? 1);
  return <SearchResults query={query} page={page} />;
}
```

Note: accessing `searchParams` makes the component dynamically rendered (opts out of static rendering).

### Route Groups

Parentheses create route groups — they organize files without affecting the URL:

```text
app/
  (marketing)/
    page.tsx       →  /
    about/
      page.tsx     →  /about
  (shop)/
    products/
      page.tsx     →  /products
```

Route groups are also used to apply different root layouts to different sections of the app without affecting URLs.

---

## 10. Dynamic Routes

### Single Segment `[param]`

```text
app/users/[id]/page.tsx → matches /users/123, /users/abc
```

```jsx
export default function Page({ params }: { params: { id: string } }) {
  return <div>User ID: {params.id}</div>;
}
```

### Catch-All `[...slug]`

Matches one or more path segments:

```text
app/docs/[...slug]/page.tsx → /docs/a, /docs/a/b, /docs/a/b/c
                              (does NOT match /docs — no segments)
```

```jsx
export default function Page({ params }: { params: { slug: string[] } }) {
  // /docs/a/b/c → params.slug = ['a', 'b', 'c']
  return <div>{params.slug.join('/')}</div>;
}
```

### Optional Catch-All `[[...slug]]`

Also matches the root segment (no slug provided):

```text
app/docs/[[...slug]]/page.tsx → /docs, /docs/a, /docs/a/b
```

```jsx
export default function Page({ params }: { params: { slug?: string[] } }) {
  // /docs → params.slug = undefined
  // /docs/a/b → params.slug = ['a', 'b']
  return <div>{params.slug?.join('/') ?? 'root'}</div>;
}
```

### Parallel Routes

Use `@folder` convention to render multiple pages simultaneously in one layout (split dashboards, modals):

```text
app/
  layout.tsx
  @team/
    page.tsx
  @analytics/
    page.tsx
  page.tsx
```

### Intercepting Routes

Use `(.)`, `(..)`, `(..)(..)` notation to intercept a route and render it in a modal context while the URL updates, without losing the current page context.

---

## 11. Layout System

Layouts in the App Router are persistent React components that wrap their children and **do not re-mount** on navigation between pages within their segment.

### Nested Layouts

```text
app/
  layout.tsx          ← RootLayout — wraps everything
  page.tsx
  dashboard/
    layout.tsx        ← DashboardLayout — wraps /dashboard/*
    page.tsx
    settings/
      layout.tsx      ← SettingsLayout — wraps /dashboard/settings/*
      page.tsx
```

On navigation from `/dashboard` to `/dashboard/settings`:
- `RootLayout` stays mounted (no unmount)
- `DashboardLayout` stays mounted (no unmount)
- `SettingsLayout` mounts fresh
- Old `page.tsx` unmounts, new `page.tsx` mounts

### Root Layout

```jsx
// app/layout.tsx
import { Inter } from 'next/font/google';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'My App',
  description: 'My Next.js application',
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={inter.className}>
      <body>
        <Header />
        <main>{children}</main>
        <Footer />
      </body>
    </html>
  );
}
```

### Layout vs Template

| | `layout.tsx` | `template.tsx` |
|---|---|---|
| Re-mounts on navigation | No | Yes |
| State preserved between pages | Yes | No |
| Use case | Persistent nav, sidebars | Per-page animations, analytics |

### Layouts Cannot Access Children's Route Params

A layout at `/users/layout.tsx` receives `params` for its own segment only. It does not have access to `[id]` from `/users/[id]/page.tsx`. The page component itself receives the full `params`.

---

## 12. Middleware

Middleware runs **before a request is completed**, at the Edge, allowing inspection and modification of the request and response before the page renders.

### Creating Middleware

```typescript
// middleware.ts (root of project, next to /app)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth-token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();  // continue to the page
}

export const config = {
  matcher: ['/dashboard/:path*', '/api/protected/:path*'],
};
```

### Matcher Configuration

```typescript
export const config = {
  matcher: [
    // Match specific paths
    '/dashboard/:path*',
    // Exclude static files and images from middleware
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
};
```

### What Middleware Can Do

- Redirect and rewrite URLs
- Add or modify response headers
- Set and read cookies
- Return a response directly (short-circuit the request)
- A/B testing based on cookies or headers
- Authentication and authorization guard

### Middleware Limitations

- Runs on the **Edge runtime** — no Node.js APIs (`fs`, native modules)
- Cannot directly access a database (use lightweight JWT verification or an Edge-compatible service)
- Adds latency to every matched request — keep it fast

---

## 13. Image Optimization

Next.js provides the `<Image>` component that automatically handles resizing, format conversion, lazy loading, and layout stability prevention.

### Basic Usage

```jsx
import Image from 'next/image';

export default function Avatar() {
  return (
    <Image
      src="/profile.jpg"
      alt="Profile picture"
      width={100}
      height={100}
    />
  );
}
```

### Key Features

- Automatic **WebP / AVIF** conversion based on browser support
- **Lazy loading** by default — images only load when near the viewport
- Prevents **Cumulative Layout Shift (CLS)** by reserving exact space before image loads
- Serves images from Next.js image optimization API (`/_next/image`)
- Supports remote images (must whitelist domains in `next.config.js`)

### Priority Loading for LCP Images

```jsx
<Image
  src="/hero.jpg"
  alt="Hero"
  width={1200}
  height={600}
  priority  // disables lazy loading and preloads — use only for above-the-fold LCP image
/>
```

### Remote Images Configuration

```javascript
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        pathname: '/images/**',
      },
    ],
  },
};
```

### next/image vs `<img>`

| | `<img>` | `<Image>` |
|---|---|---|
| Format conversion | No | Yes (WebP/AVIF) |
| Lazy loading | No (manual `loading` attr) | Yes (automatic) |
| Prevents layout shift | No | Yes (reserves dimensions) |
| Responsive sizing | Manual | `sizes` prop + srcset |
| CDN optimization | No | Yes via `/_next/image` |

---

## 14. API Routes

### App Router: route.ts

```typescript
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const limit = searchParams.get('limit') ?? '10';
  const users = await db.user.findMany({ take: Number(limit) });
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  const user = await db.user.create({ data: body });
  return NextResponse.json(user, { status: 201 });
}
```

### Dynamic Route Handler

```typescript
// app/api/users/[id]/route.ts
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await db.user.findUnique({ where: { id: params.id } });
  if (!user) return NextResponse.json({ error: 'Not found' }, { status: 404 });
  return NextResponse.json(user);
}

export async function DELETE(
  request: Request,
  { params }: { params: { id: string } }
) {
  await db.user.delete({ where: { id: params.id } });
  return new Response(null, { status: 204 });
}
```

### Pages Router: /pages/api

```typescript
// pages/api/users.ts
import type { NextApiRequest, NextApiResponse } from 'next';

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === 'GET') {
    res.status(200).json({ users: [] });
  } else if (req.method === 'POST') {
    res.status(201).json({ created: true });
  } else {
    res.setHeader('Allow', ['GET', 'POST']);
    res.status(405).end(`Method ${req.method} Not Allowed`);
  }
}
```

### When to Use API Routes vs Server Actions

| Scenario | Recommended |
|---|---|
| Third-party webhooks (Stripe, GitHub) | API route (`route.ts`) |
| Mobile app consuming the API | API route (`route.ts`) |
| Form submissions from Next.js UI | Server Action |
| Mutations triggered from buttons in Next.js | Server Action |
| Public REST API for external consumers | API route (`route.ts`) |

---

## 15. Next.js Performance Features

### Automatic Code Splitting

Every route in Next.js is automatically code-split. JavaScript for `/users` is not loaded when visiting `/`. This reduces initial bundle size without any configuration.

### next/font

```jsx
// app/layout.tsx
import { Inter, Roboto_Mono } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
});

export default function RootLayout({ children }) {
  return (
    <html className={inter.variable}>
      <body>{children}</body>
    </html>
  );
}
```

- Fonts are downloaded at **build time** and self-hosted — no runtime DNS lookup to Google
- Eliminates layout shift from font loading
- Supports `font-display: swap` and custom fonts via `next/font/local`

### next/script

```jsx
import Script from 'next/script';

// strategy: beforeInteractive | afterInteractive | lazyOnload | worker
<Script
  src="https://analytics.example.com/script.js"
  strategy="afterInteractive"  // loads after page becomes interactive
/>
```

### Streaming with Suspense

Wrapping slow components in `<Suspense>` allows Next.js to send the rest of the page immediately while the suspended component is still resolving. This improves Time to First Byte (TTFB) and perceived performance significantly.

### Turbopack (Next.js 14+)

The new Rust-based bundler replaces Webpack for the dev server:

```bash
next dev --turbo
```

Provides dramatically faster cold starts and Hot Module Replacement in large projects.

---

## 16. Deployment

### Vercel (Zero Configuration)

```bash
npm install -g vercel
vercel
```

All Next.js features work automatically on Vercel: ISR, Edge Middleware, Image Optimization, Streaming, Edge Functions.

### Self-Hosted Node.js

```bash
next build
next start  # starts Node.js server on port 3000 by default
```

Requires a Node.js server. ISR and Image Optimization work. Edge Middleware requires additional configuration.

### Static Export

```javascript
// next.config.js
module.exports = {
  output: 'export',
};
```

```bash
next build
# Outputs to /out directory — pure static files, no server required
```

**Limitations of static export:**
- No SSR (no per-request server processing)
- No ISR
- No Image Optimization API
- No API routes
- No Middleware

Use for: pure static sites hosted on S3, GitHub Pages, Netlify, or any static CDN.

### Environment Variables

```bash
# .env.local
DATABASE_URL=postgres://...          # server-only, never sent to browser
NEXT_PUBLIC_API_URL=https://api...   # prefixed with NEXT_PUBLIC_ → exposed to browser
```

```javascript
process.env.DATABASE_URL         // server-side only
process.env.NEXT_PUBLIC_API_URL  // available on both server and client
```

---

## 17. Common Interview Questions

### Q: When would you use SSR vs SSG?

**SSG:** Content that is the same for all users and does not change frequently — marketing pages, blog posts, documentation. Delivered from CDN with maximum speed.

**SSR:** Content that is user-specific (requires auth context) or must be real-time accurate — dashboards, personalized feeds, financial data. Higher server cost but always fresh and personalized.

**ISR:** The middle ground — public content that changes occasionally. A blog with updated content, product catalog.

### Q: What are React Server Components and how does Next.js use them?

RSC is a React architecture where components run only on the server. Their serialized render output (not HTML — a React tree description) is sent to the client. The client receives pre-rendered content without the component's JavaScript. Next.js App Router makes all components RSC by default, enabling direct database access, reduced bundle sizes, and async component patterns without any hooks.

### Q: How do you share state between Server and Client Components?

Server Components cannot receive state from the client. You share data **downward** (server to client) only:
- Server Component fetches data and passes it as **props** to a Client Component
- Client Component manages its own interactive state independently
- Persistent shared state goes through **URL** (search params), **cookies**, or **database** — not React state

### Q: What is hydration and what causes hydration errors?

Hydration is the process where React attaches event listeners to server-rendered HTML and makes the page interactive in the browser. React renders the component tree in the browser and compares it to the server-rendered HTML. If they differ, a **hydration mismatch error** occurs and React re-renders from scratch.

Common causes:
- `Date.now()` or `Math.random()` called during render (different values each time)
- `typeof window !== 'undefined'` branches rendering different content server vs client
- Using browser-only APIs without guards during initial render
- `suppressHydrationWarning` can silence intentional differences (like client-side timestamps)

### Q: What is the difference between layout.tsx and template.tsx?

`layout.tsx` persists across navigations — it does not unmount. State preserved inside a layout survives navigation between sibling pages.

`template.tsx` creates a new instance on every navigation — it unmounts and remounts. Useful for enter/exit animations or per-page event tracking.

---

## 18. Common Mistakes

### Using useState in a Server Component

❌ Wrong:
```jsx
// app/counter/page.tsx — Server Component
import { useState } from 'react';  // Error

export default function CounterPage() {
  const [count, setCount] = useState(0);  // Runtime error in App Router
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

✅ Correct:
```jsx
'use client';
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### Forgetting "use client" When Using Hooks

❌ Wrong:
```jsx
// app/components/ThemeToggle.tsx — missing "use client"
import { useState } from 'react';

export default function ThemeToggle() {
  const [dark, setDark] = useState(false);  // Error: hooks not allowed in Server Component
}
```

✅ Correct:
```jsx
'use client';

import { useState } from 'react';

export default function ThemeToggle() {
  const [dark, setDark] = useState(false);
  // ...
}
```

### Fetching in a Client Component When Server Component Would Work

❌ Wrong (unnecessary client-side fetch):
```jsx
'use client';
import { useState, useEffect } from 'react';

export default function PostList() {
  const [posts, setPosts] = useState([]);
  useEffect(() => {
    fetch('/api/posts').then(r => r.json()).then(setPosts);
  }, []);
  // Issues: no caching, waterfall loading, exposes fetch to client, no SEO
}
```

✅ Correct:
```jsx
// app/posts/page.tsx — Server Component, no "use client" needed
export default async function PostList() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>;
}
```

### Importing Heavy Client Libraries in Server Components

❌ Wrong:
```jsx
// Server Component — browser-only library will crash or bloat server bundle
import { Chart } from 'heavy-chart-library';
```

✅ Correct:
```jsx
'use client';
import { Chart } from 'heavy-chart-library';  // stays in Client Component
```

---

## 19. Best Practices

### 1. Maximize Server Components

Push `"use client"` as far down the component tree as possible. Only the interactive leaf nodes need to be Client Components. Server Components cost zero JS bundle.

```text
Page (Server)
  └── Layout (Server)
        ├── StaticHeader (Server)
        ├── DataList (Server)
        │     └── ListItem (Server)
        └── InteractiveButton (Client)  ← "use client" only here
```

### 2. Use Streaming for Slow Data

Do not block the entire page for slow data. Wrap slow Server Components in `<Suspense>`:

```jsx
export default function Dashboard() {
  return (
    <>
      <FastStats />                   {/* renders immediately */}
      <Suspense fallback={<Spinner />}>
        <SlowReports />               {/* streams in when ready */}
      </Suspense>
    </>
  );
}
```

### 3. Use Parallel Data Fetching

```jsx
// Always prefer Promise.all for independent requests
const [user, orders] = await Promise.all([fetchUser(id), fetchOrders(id)]);
```

### 4. Always Use next/image and next/font

Use `<Image>` instead of `<img>` and `next/font` instead of `@import` for fonts. These are zero-effort, high-impact performance wins built into Next.js.

### 5. Use Tag-Based Revalidation for Precise Cache Control

```jsx
// Fetch with a tag
fetch('/api/posts', { next: { tags: ['posts'] } });

// In a Server Action, invalidate only what changed
import { revalidateTag } from 'next/cache';
revalidateTag('posts');  // only fetches tagged 'posts' are invalidated
```

### 6. Use TypeScript for Params and SearchParams

```typescript
type PageProps = {
  params: { id: string };
  searchParams: { page?: string; sort?: string };
};

export default function Page({ params, searchParams }: PageProps) {
  // ...
}
```

### 7. Keep Secrets Server-Side

Never use environment variables without `NEXT_PUBLIC_` prefix in Client Components — they will be `undefined` at runtime. Always access secrets in Server Components, Server Actions, or API routes.

---
