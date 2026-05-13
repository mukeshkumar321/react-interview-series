# Vite

## Table of Contents

1. [What is Vite and Why It Is Faster](#1-what-is-vite-and-why-it-is-faster)
2. [How the Vite Dev Server Works](#2-how-the-vite-dev-server-works)
3. [How the Vite Production Build Works](#3-how-the-vite-production-build-works)
4. [Creating a React Project with Vite](#4-creating-a-react-project-with-vite)
5. [vite.config.js and vite.config.ts](#5-viteconfigjs-and-viteconfigts)
6. [Environment Variables](#6-environment-variables)
7. [Import Aliases](#7-import-aliases)
8. [CSS Handling](#8-css-handling)
9. [Asset Handling](#9-asset-handling)
10. [HMR — Hot Module Replacement](#10-hmr--hot-module-replacement)
11. [Plugins Ecosystem](#11-plugins-ecosystem)
12. [Proxy Configuration](#12-proxy-configuration)
13. [Build Optimization](#13-build-optimization)
14. [CRA vs Vite Migration](#14-cra-vs-vite-migration)
15. [Common Mistakes and Gotchas](#15-common-mistakes-and-gotchas)
16. [Best Practices](#16-best-practices)

---

## 1. What is Vite and Why It Is Faster

Vite (French for "fast") is a modern frontend build tool created by Evan You (creator of Vue.js). It provides a **development server** and a **production build pipeline** designed to eliminate the startup and reload bottlenecks of traditional bundler-based tools like Webpack and Create React App.

### The Problem with CRA / Webpack in Development

Traditional bundlers like Webpack must process the **entire application** before serving anything:

```text
Webpack dev server startup:
─────────────────────────────────────
1. Parse every JS/TS/JSX file
2. Build a full dependency graph
3. Transform (Babel, TypeScript)
4. Bundle into chunks
5. Start dev server
─────────────────────────────────────
Time for large app: 30s–120s
```

This makes initial startup slow, and full re-bundling on every save makes HMR increasingly slow as the app grows.

### How Vite Is Different

Vite uses two separate approaches for development vs production:

```text
Vite dev server:
─────────────────────────────────────
1. Start dev server instantly (< 300ms)
2. Serve files as native ES modules
3. Only transform files when requested
4. Transform one module at a time, on demand
─────────────────────────────────────
HMR: updates only the changed module, ~10ms
```

### Key Speed Advantage: Native ES Modules

Modern browsers support native ES module imports (`<script type="module">`). Vite exploits this:

```text
CRA / Webpack:                    Vite Dev Server:
──────────────────────────────    ──────────────────────────────
All code bundled into             Browser requests files
one or a few large bundles        individually as ES modules

Browser loads bundle.js           Browser: GET /src/main.tsx
(must bundle first)               Vite: transform and serve
                                  Browser: GET /src/components/App.tsx
                                  Vite: transform and serve
                                  (only transforms what is needed)
```

### Pre-Bundling of Dependencies

Vite still bundles `node_modules` dependencies using **esbuild** (not Rollup) — once, at server start, and caches them. This is called **dependency pre-bundling**.

- **Why:** npm packages use CommonJS (CJS) which browsers cannot natively import
- **How:** esbuild converts CJS → ESM, bundles each package into a single ESM file
- **Result:** Cached in `.vite/deps/` — only re-runs when dependencies change

```text
First start:
└── Pre-bundle: react, react-dom, lodash → .vite/deps/*.js (fast esbuild)

Subsequent starts:
└── Serve pre-bundled deps from cache (instant)
```

### Vite vs CRA vs Webpack Summary

| Dimension | CRA (Webpack) | Vite |
|---|---|---|
| Dev server start | 15s–120s | < 1s |
| HMR speed | Slow (full bundle) | Near-instant (single module) |
| Build tool (dev) | Webpack | Native ESM + esbuild |
| Build tool (prod) | Webpack | Rollup |
| Config | Ejected or craco | vite.config.ts |
| TypeScript support | Via Babel (no type check) | Via esbuild (no type check) |
| Tree shaking | Webpack (good) | Rollup (excellent) |
| Bundle size | Larger | Smaller (better tree shaking) |
| Actively maintained | Deprecated by Meta | Active, industry standard |

---

## 2. How the Vite Dev Server Works

### Native ESM Pipeline

When you run `vite` (or `npm run dev`), Vite:

1. Starts an HTTP server (default port 5173)
2. Pre-bundles `node_modules` with esbuild
3. Injects a client script into `index.html` for HMR
4. Waits for browser requests

When the browser requests a file:

```text
Browser requests: /src/App.tsx
         ↓
Vite receives request
         ↓
Reads App.tsx from disk
         ↓
Transforms: TypeScript → JS, JSX → JS (using esbuild)
         ↓
Injects HMR runtime code
         ↓
Returns transformed module as text/javascript
         ↓
Browser executes the module
Browser makes new request for each import in App.tsx
```

### Request-Time Transformation

Each file is transformed **only when the browser requests it**. If `ComponentX` is never imported, it is never transformed. This is why cold start is fast even in large projects.

```text
App.tsx imports: Header, Footer, Dashboard
Browser requests only those three → only those three are transformed
Dozens of other components never requested → never transformed
```

### Source Maps

Vite generates source maps automatically in development. When you see an error in the browser, the stack trace points to your original TypeScript/JSX source file — not the transformed JavaScript.

---

## 3. How the Vite Production Build Works

The production build uses **Rollup** (not esbuild) as the bundler. Rollup provides:
- Superior tree shaking (dead code elimination)
- Advanced code splitting
- Mature plugin ecosystem

### Build Pipeline

```bash
npm run build
# Runs: vite build
```

```text
vite build:
1. Parse all modules starting from index.html
2. Rollup builds a dependency graph
3. Tree shake unused exports
4. Split code into chunks (per route, per vendor)
5. TypeScript errors checked (via tsc if configured)
6. Output to dist/ directory:
   dist/
     index.html       (with hashed asset links)
     assets/
       index-abc123.js   (main bundle, hashed)
       vendor-def456.js  (node_modules, hashed)
       index-ghi789.css  (styles, hashed)
       logo-jkl012.png   (assets, hashed)
```

### Content Hash in Filenames

Vite appends a content hash to all asset filenames. This enables **long-term browser caching** — when code changes, the hash changes, so browsers fetch fresh files. Unchanged files keep the same hash and stay cached.

```html
<!-- Generated index.html -->
<script type="module" src="/assets/index-a1b2c3d4.js"></script>
<link rel="stylesheet" href="/assets/index-e5f6g7h8.css">
```

### Preview Built Output

```bash
npm run preview
# Serves the dist/ folder locally to test the production build
```

---

## 4. Creating a React Project with Vite

```bash
# Interactive scaffolding
npm create vite@latest

# Non-interactive — specify framework and variant
npm create vite@latest my-app -- --template react-ts

# Available templates:
# vanilla, vanilla-ts, vue, vue-ts, react, react-ts, react-swc, react-swc-ts
# preact, lit, svelte, svelte-ts, solid, solid-ts, qwik, qwik-ts
```

### Generated Project Structure

```text
my-app/
  ├── public/
  │     └── vite.svg          ← static assets (served as-is at root URL)
  ├── src/
  │     ├── assets/
  │     │     └── react.svg   ← imported assets (hashed in build)
  │     ├── App.tsx
  │     ├── App.css
  │     ├── main.tsx          ← entry point
  │     └── vite-env.d.ts     ← TypeScript types for Vite globals
  ├── index.html              ← HTML entry (not in public/)
  ├── package.json
  ├── tsconfig.json
  ├── tsconfig.node.json
  └── vite.config.ts
```

### Key Difference: index.html Location

In CRA, `index.html` is in `public/`. In Vite, `index.html` is at the **project root** and is the actual entry point. Vite processes it directly.

```html
<!-- index.html — Vite treats this as the entry -->
<!DOCTYPE html>
<html>
  <head><title>My App</title></head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
    <!-- ↑ native ESM script — Vite serves this file -->
  </body>
</html>
```

---

## 5. vite.config.js and vite.config.ts

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  // ─── Plugins ────────────────────────────────────────────────────────────────
  plugins: [
    react(),  // JSX transform + Fast Refresh HMR
  ],

  // ─── Resolve ────────────────────────────────────────────────────────────────
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),     // @ → ./src
      '@components': path.resolve(__dirname, './src/components'),
    },
  },

  // ─── Dev Server ─────────────────────────────────────────────────────────────
  server: {
    port: 3000,           // default is 5173
    open: true,           // auto-open browser
    host: true,           // expose to local network (0.0.0.0)
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },

  // ─── Build ──────────────────────────────────────────────────────────────────
  build: {
    outDir: 'dist',          // default
    sourcemap: true,         // generate source maps for production
    minify: 'esbuild',       // 'esbuild' (fast) | 'terser' (smaller) | false
    target: 'esnext',        // browser targets for transpilation
    chunkSizeWarningLimit: 500,  // warn if chunk > 500kB
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],   // separate vendor chunk
        },
      },
    },
  },

  // ─── CSS ────────────────────────────────────────────────────────────────────
  css: {
    modules: {
      localsConvention: 'camelCase',  // import styles.myClass (not styles['my-class'])
    },
    preprocessorOptions: {
      scss: {
        additionalData: `@import "@/styles/variables.scss";`,  // auto-import Sass vars
      },
    },
  },

  // ─── Test (Vitest) ──────────────────────────────────────────────────────────
  test: {
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    globals: true,
  },
});
```

---

## 6. Environment Variables

Vite uses `.env` files and exposes variables to the browser using a **strict naming convention**.

### The VITE_ Prefix Rule

```bash
# .env
VITE_API_URL=https://api.example.com   # ✅ exposed to browser
VITE_APP_TITLE=My App                  # ✅ exposed to browser
DATABASE_PASSWORD=secret123            # ❌ NOT exposed — no VITE_ prefix
JWT_SECRET=mysecret                    # ❌ NOT exposed — no VITE_ prefix
```

**Only variables prefixed with `VITE_` are embedded into the client bundle.** Others are completely ignored at runtime. This prevents accidentally exposing server secrets.

### Accessing Variables

```ts
// In client code (browser)
const apiUrl = import.meta.env.VITE_API_URL;  // ✅
const title = import.meta.env.VITE_APP_TITLE;  // ✅
const dbPass = import.meta.env.DATABASE_PASSWORD;  // undefined — not exposed

// Built-in Vite variables (always available)
import.meta.env.MODE     // 'development' | 'production' | 'test'
import.meta.env.DEV      // true in development
import.meta.env.PROD     // true in production
import.meta.env.SSR      // true in SSR mode
import.meta.env.BASE_URL // base URL (set via 'base' config option)
```

### CRA vs Vite Environment Variables

```ts
// CRA
process.env.REACT_APP_API_URL    // requires REACT_APP_ prefix

// Vite
import.meta.env.VITE_API_URL     // requires VITE_ prefix + import.meta.env
```

### .env File Hierarchy

Files are loaded in this priority order (higher = higher priority):

```text
.env.local              ← always loaded, highest priority, git-ignored
.env.[mode].local       ← .env.production.local, .env.development.local
.env.[mode]             ← .env.production, .env.development, .env.test
.env                    ← always loaded, lowest priority
```

```bash
# .env           — defaults for all environments
# .env.local     — local overrides (in .gitignore)
# .env.development — only in development
# .env.production  — only in production build

VITE_API_URL=https://api.example.com     # .env
VITE_API_URL=http://localhost:3001       # .env.development (overrides .env in dev)
```

### TypeScript Types for env vars

```ts
// vite-env.d.ts (or src/env.d.ts)
/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string;
  readonly VITE_APP_TITLE: string;
  // add all your VITE_ variables here
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

---

## 7. Import Aliases

Aliases eliminate relative path chains (`../../../components/Button`).

### Setup

```ts
// vite.config.ts
import path from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### TypeScript Configuration

Also update `tsconfig.json` so TypeScript recognizes the alias:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

### Usage

```ts
// ❌ Before — fragile relative paths
import Button from '../../../components/ui/Button';
import { useAuth } from '../../hooks/useAuth';

// ✅ After — clean alias paths
import Button from '@/components/ui/Button';
import { useAuth } from '@/hooks/useAuth';
```

---

## 8. CSS Handling

### Plain CSS

```ts
// Import CSS in your component or entry file
import './App.css';
import styles from './App.module.css';  // CSS Modules
```

### CSS Modules

Files named `*.module.css` (or `.module.scss`) are automatically treated as CSS Modules:

```css
/* Button.module.css */
.primary { background: blue; color: white; }
.large { padding: 12px 24px; }
```

```tsx
import styles from './Button.module.css';

export function Button({ children }) {
  return <button className={styles.primary}>{children}</button>;
}
// Rendered class: "Button_primary__abc123" (scoped hash)
```

### PostCSS

Create a `postcss.config.cjs` at the project root — Vite picks it up automatically:

```js
// postcss.config.cjs
module.exports = {
  plugins: {
    'autoprefixer': {},
    'postcss-nesting': {},
  },
};
```

### Sass/SCSS

Install the preprocessor — no plugin needed:

```bash
npm install -D sass
```

```scss
/* styles.scss */
$primary: #3b82f6;

.button {
  background: $primary;
  &:hover { background: darken($primary, 10%); }
}
```

```ts
import './styles.scss';  // Vite handles SCSS automatically
```

---

## 9. Asset Handling

### Importing Assets

```ts
import logoUrl from './assets/logo.png';     // returns URL string
import logoRaw from './assets/logo.svg?raw'; // returns SVG source string
import data from './data.json';              // JSON imported as object

// Usage
<img src={logoUrl} />       // /assets/logo-abc123.png (hashed in build)
```

### The `public/` Folder

Files in `public/` are served at the root URL **as-is** — no processing, no hashing:

```text
public/
  favicon.ico       → /favicon.ico
  robots.txt        → /robots.txt
  og-image.png      → /og-image.png  (URL is stable)
```

Use `public/` for files that:
- Must have a stable, predictable URL (favicon, robots.txt, manifests)
- Are referenced from HTML directly (not imported in JS/TS)
- Are loaded by external services that need a fixed URL

### Import URL Variants

```ts
// URL of asset (default for images/fonts/etc.)
import myImage from './image.png';  // → '/assets/image-abc.png'

// Inline as data URI (for small assets)
import tinyIcon from './icon.svg?inline';  // → 'data:image/svg+xml,...'

// Raw string content
import svgSource from './icon.svg?raw';  // → '<svg>...</svg>'

// URL explicitly (even for JS files — useful for workers)
import workerUrl from './worker.ts?url';  // → '/assets/worker-abc.js'
import WorkerConstructor from './worker.ts?worker';  // → Web Worker class
```

### Dynamic Imports

```ts
// Lazy load a component (code splitting)
const LazyChart = React.lazy(() => import('./Chart'));

// Dynamic asset loading
const moduleUrl = await import('./data.json');
```

---

## 10. HMR — Hot Module Replacement

HMR updates the browser without a full page reload, preserving application state.

### How Vite HMR Works

```text
Developer saves App.tsx
     ↓
Vite server detects file change (file system watcher)
     ↓
Vite re-transforms only App.tsx
     ↓
Vite sends a WebSocket message to the browser:
  { type: 'update', updates: [{ path: '/src/App.tsx', ... }] }
     ↓
Browser HMR client receives the message
     ↓
Browser re-imports the updated module
     ↓
React Fast Refresh re-renders the component in place
     ↓
State is preserved — no full page reload
```

### React Fast Refresh

`@vitejs/plugin-react` includes **React Fast Refresh** — React's official HMR solution. It:

- Preserves component state on re-render
- Remounts components if hooks change (to avoid inconsistency)
- Falls back to full page reload only if necessary

```ts
// vite.config.ts
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],  // Fast Refresh included automatically
});
```

### Manual HMR API

For non-React code (stores, utilities), use the HMR API:

```ts
// store.ts
export let store = createStore();

if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    // Re-use new module exports
    store = newModule?.store ?? store;
  });

  import.meta.hot.dispose(() => {
    // Cleanup before module is replaced
    store.destroy();
  });
}
```

---

## 11. Plugins Ecosystem

### Official Vite Plugins

```bash
# React with Babel (default — supports styled-components, emotion)
npm install -D @vitejs/plugin-react

# React with SWC (Rust-based, 20x faster transforms than Babel)
npm install -D @vitejs/plugin-react-swc
```

```ts
// Babel (default — best compatibility)
import react from '@vitejs/plugin-react';

// SWC (faster, but fewer Babel plugin options)
import react from '@vitejs/plugin-react-swc';
```

### When to Use Which

| Plugin | Transforms with | Speed | Compatibility |
|---|---|---|---|
| `@vitejs/plugin-react` | Babel | Moderate | Full Babel ecosystem |
| `@vitejs/plugin-react-swc` | SWC (Rust) | 20x faster | Limited Babel plugins |

**Use `plugin-react-swc`** for most projects — it's significantly faster with no practical limitations for standard React.

**Use `plugin-react`** when you need Babel plugins like `babel-plugin-styled-components`, `babel-plugin-emotion`, or `babel-plugin-macros`.

### Common Community Plugins

```bash
vite-plugin-svgr        # Import SVGs as React components
vite-tsconfig-paths     # Automatic tsconfig path alias support
rollup-plugin-visualizer # Bundle size visualization
vite-plugin-pwa         # Progressive Web App support
unplugin-icons          # Import icon libraries as components
```

---

## 12. Proxy Configuration

In development, the browser cannot make cross-origin requests to your backend API without CORS. The proxy forwards requests from the dev server to avoid this.

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      // Simple string proxy
      '/api': 'http://localhost:8080',

      // Advanced proxy with options
      '/auth': {
        target: 'http://localhost:8080',
        changeOrigin: true,       // change origin header to target
        secure: false,            // allow self-signed certs in dev
        rewrite: (path) => path.replace(/^\/auth/, '/api/auth'),
      },

      // WebSocket proxy
      '/ws': {
        target: 'ws://localhost:8080',
        ws: true,
      },
    },
  },
});
```

### How Proxy Works

```text
Browser: fetch('/api/users')
           ↓
Vite dev server receives request
           ↓
Proxy: forwards to http://localhost:8080/api/users
           ↓
Backend: responds with JSON
           ↓
Vite: forwards response back to browser
           ↓
Browser: receives response (same origin — no CORS issue)
```

This only applies in **development**. In production, your deployment infrastructure (Nginx, reverse proxy, API gateway) handles this.

---

## 13. Build Optimization

### Manual Chunks

Split vendor dependencies into separate chunks for better caching:

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          // React core in its own chunk
          vendor: ['react', 'react-dom'],
          // Router in its own chunk
          router: ['react-router-dom'],
          // Charts separated (heavy dependency)
          charts: ['recharts'],
        },
      },
    },
  },
});
```

### Code Splitting with Dynamic Imports

```tsx
import { lazy, Suspense } from 'react';

// Split each route into a separate chunk
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Reports = lazy(() => import('./pages/Reports'));

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

### Visualize Bundle

```bash
npm install -D rollup-plugin-visualizer
```

```ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({ open: true }),  // opens treemap in browser after build
  ],
});
```

### Build Targets

```ts
export default defineConfig({
  build: {
    target: 'esnext',   // modern browsers — smallest output
    // target: 'es2015', // IE11 support — larger output
    // target: ['chrome87', 'firefox78', 'safari14'],  // specific browsers
  },
});
```

---

## 14. CRA vs Vite Migration

### Step-by-Step Migration

```bash
# 1. Uninstall CRA dependencies
npm uninstall react-scripts

# 2. Install Vite
npm install -D vite @vitejs/plugin-react

# 3. Create vite.config.ts
# 4. Move index.html from public/ to root
# 5. Add <script type="module" src="/src/index.tsx"> to index.html
# 6. Replace %PUBLIC_URL% references in index.html
# 7. Rename REACT_APP_ env vars to VITE_
# 8. Replace process.env.REACT_APP_X with import.meta.env.VITE_X
# 9. Update package.json scripts
```

### Script Changes

```json
// CRA (package.json)
{
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test"
  }
}

// Vite (package.json)
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
}
```

### Key Differences During Migration

| CRA | Vite |
|---|---|
| `process.env.REACT_APP_X` | `import.meta.env.VITE_X` |
| `public/index.html` | `index.html` (root) |
| No config needed | `vite.config.ts` required |
| `react-scripts test` (Jest) | Vitest (or configure Jest with jsdom) |
| `%PUBLIC_URL%` | Use absolute path or base URL |
| All deps bundled | `node_modules` pre-bundled with esbuild |

---

## 15. Common Mistakes and Gotchas

### Using process.env Instead of import.meta.env

```ts
// ❌ Wrong — process.env does not exist in Vite's browser environment
const apiUrl = process.env.REACT_APP_API_URL;  // undefined

// ✅ Correct
const apiUrl = import.meta.env.VITE_API_URL;
```

### Forgetting the VITE_ Prefix

```bash
# .env
API_SECRET=abc123         # ❌ Never exposed to browser (intentional security)
VITE_API_URL=https://...  # ✅ Exposed with prefix
```

```ts
import.meta.env.API_SECRET  // undefined — not exposed
import.meta.env.VITE_API_URL  // 'https://...' — exposed
```

### CommonJS Modules Without Pre-Bundling

Some npm packages are CommonJS-only and may cause issues if not pre-bundled:

```ts
// vite.config.ts
export default defineConfig({
  optimizeDeps: {
    include: ['some-cjs-package'],  // Force pre-bundling of CJS packages
  },
});
```

### Dynamic require() in Browser Code

```ts
// ❌ Wrong — require() is not available in Vite's native ESM environment
const config = require('./config');

// ✅ Correct — use static or dynamic import
import config from './config';
const config = await import('./config');
```

### Forgetting tsconfig paths Alongside vite.config Aliases

Setting an alias in `vite.config.ts` handles build-time resolution. TypeScript's `paths` in `tsconfig.json` handles editor type checking. Both must be set.

### Referencing Public Folder Assets

```ts
// ❌ Wrong — do not import from public/ (they are served as static files)
import logo from '/public/logo.png';

// ✅ Correct — reference with absolute URL
<img src="/logo.png" />  // public/logo.png is available at /logo.png

// ✅ Or import from src/assets/ (will be processed and hashed)
import logo from './assets/logo.png';
```

---

## 16. Best Practices

### Use TypeScript and vite.config.ts

Always use TypeScript for your Vite config — you get full autocomplete and type safety for all options.

### Organize Environment Variables

```bash
.env                # base defaults (committed)
.env.local          # local dev secrets (git-ignored)
.env.production     # production non-secrets (committed)
```

### Always Define TypeScript Types for Env Vars

Extend `ImportMetaEnv` in `vite-env.d.ts` for full type safety on all `import.meta.env.*` accesses.

### Lazy-Load Routes

Every page component should be lazy-loaded with `React.lazy()` and `<Suspense>`. This creates separate chunks per route and dramatically reduces initial load time.

### Separate Vendor and App Code

Use `manualChunks` to put `react`, `react-dom`, and other stable dependencies in a vendor chunk. This chunk rarely changes and can be cached by browsers long-term.

### Use plugin-react-swc for New Projects

```bash
npm create vite@latest my-app -- --template react-swc-ts
```

SWC is significantly faster than Babel with no practical downsides for standard React projects.

### Enable Source Maps in Production Selectively

```ts
build: {
  sourcemap: true,  // Full source maps — good for debugging, exposes source
  // sourcemap: 'hidden',  // No link in bundle, but maps exist for error reporting
}
```

### Summary Reference

| Concern | Recommendation |
|---|---|
| Dev server speed | Native ESM — no configuration needed |
| React plugin | `plugin-react-swc` for new projects |
| Env variables | `VITE_` prefix + `import.meta.env` |
| Aliases | Set in both `vite.config.ts` and `tsconfig.json` |
| CSS | CSS Modules for component styles, global CSS for base/reset |
| Code splitting | `React.lazy()` + `<Suspense>` per route |
| Bundle analysis | `rollup-plugin-visualizer` |
| API in dev | `server.proxy` configuration |
| CJS packages | Add to `optimizeDeps.include` |
| Static assets | `public/` for stable URLs, `src/assets/` for imported assets |
