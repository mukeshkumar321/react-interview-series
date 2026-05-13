## Vite — Tricky Output Questions

> These questions test deep understanding of Vite's dev server behavior, build output, environment variable handling, configuration, and edge cases. Each question reflects real interview and debugging scenarios.

---

## 1. Dev Server

### Q1

```ts
// src/api.ts
const baseUrl = process.env.REACT_APP_API_URL;

export function fetchUser(id: string) {
  return fetch(`${baseUrl}/users/${id}`).then(r => r.json());
}
```

This file is in a Vite project (not CRA). The `.env` file has:
```
REACT_APP_API_URL=https://api.example.com
```

#### ❓ What is the value of `baseUrl` at runtime in the browser?

<details>
<summary>✅ Answer</summary>

```txt
undefined
```

**Explanation:** Vite does not replace `process.env.*` variables. Vite only exposes variables through `import.meta.env.*`. Additionally, only variables prefixed with `VITE_` are exposed to the browser bundle. `REACT_APP_API_URL` has no `VITE_` prefix and `process.env` is not a valid way to access Vite env variables. The fix requires both renaming the variable to `VITE_API_URL` and accessing it via `import.meta.env.VITE_API_URL`.

</details>

---

### Q2

```ts
// .env file
VITE_API_URL=https://api.example.com
DB_PASSWORD=supersecret123
VITE_APP_MODE=production
```

```ts
// src/config.ts
console.log(import.meta.env.VITE_API_URL);
console.log(import.meta.env.DB_PASSWORD);
console.log(import.meta.env.VITE_APP_MODE);
```

#### ❓ What does each `console.log` print in the browser?

<details>
<summary>✅ Answer</summary>

```txt
https://api.example.com  ← VITE_ prefix → exposed
undefined                 ← no VITE_ prefix → NOT exposed (security)
production               ← VITE_ prefix → exposed
```

**Explanation:** Vite's security model: only variables prefixed with `VITE_` are embedded into the client bundle. `DB_PASSWORD` lacks the `VITE_` prefix and is deliberately excluded — even if you know the variable name, the browser bundle simply doesn't contain it. This prevents accidental exposure of secrets. The `VITE_` prefix acts as an explicit opt-in for client exposure.

</details>

---

### Q3

```ts
// src/App.tsx
console.log(import.meta.env.MODE);
console.log(import.meta.env.DEV);
console.log(import.meta.env.PROD);
```

This runs in development (`npm run dev`). What is logged?

#### ❓ What are the values of these built-in Vite env variables in development?

<details>
<summary>✅ Answer</summary>

```txt
development  ← MODE
true          ← DEV
false         ← PROD
```

**Explanation:** Vite always provides these built-in environment variables regardless of your `.env` files. `MODE` is `'development'` during `vite` (dev) and `'production'` during `vite build`. `DEV` is `true` only in development mode. `PROD` is `true` only in production builds. These can be used for conditional behavior: `if (import.meta.env.DEV) { enableDebugMode(); }`.

</details>

---

### Q4

```ts
// src/App.tsx
const apiUrl = import.meta.env.VITE_API_URL;

// This is the only usage of VITE_API_URL in the entire codebase
```

The production build runs with:
```bash
VITE_API_URL=https://api.example.com npm run build
```

#### ❓ Where is `VITE_API_URL` embedded in the build output? Can it be easily found?

<details>
<summary>✅ Answer</summary>

```txt
Yes — VITE_API_URL is statically inlined into the JavaScript bundle.
The minified output will contain the literal string "https://api.example.com".

// In the minified bundle:
const a="https://api.example.com";
```

**Explanation:** During `vite build`, `import.meta.env.VITE_*` variables are statically replaced with their values — similar to a find-and-replace. The string `https://api.example.com` appears in plain text in the bundle. Anyone who downloads your JS file can read it. This is expected for public API URLs. **Never** put actual secrets (API keys, passwords) in `VITE_*` variables — they will be visible in the bundle.

</details>

---

### Q5

```ts
// .env
VITE_API_URL=https://default.example.com

// .env.development
VITE_API_URL=http://localhost:3001

// .env.local  
VITE_API_URL=http://localhost:9999
```

Developer runs `npm run dev`. What value does `import.meta.env.VITE_API_URL` have?

#### ❓ Which `.env` file wins and what is the final value?

<details>
<summary>✅ Answer</summary>

```txt
http://localhost:9999
```

**Explanation:** Vite loads `.env` files in priority order (highest to lowest):
1. `.env.[mode].local` (e.g., `.env.development.local`)
2. `.env.local` ← wins here
3. `.env.[mode]` (e.g., `.env.development`)
4. `.env`

`.env.local` has the highest priority among the loaded files. It is intended for local developer overrides and is included in `.gitignore` by default. Both `.env.development` and `.env` are overridden by `.env.local`.

</details>

---

## 2. Build Output

### Q6

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
        },
      },
    },
  },
});
```

What does the `dist/assets/` directory contain after `npm run build`?

#### ❓ What chunks are generated and why?

<details>
<summary>✅ Answer</summary>

```txt
dist/assets/
  index-[hash].js     ← application code
  vendor-[hash].js    ← react + react-dom (manualChunks)
  index-[hash].css    ← all extracted styles
```

**Explanation:** `manualChunks` tells Rollup to bundle `react` and `react-dom` into a separate chunk named `vendor`. This is a production best practice: the vendor chunk rarely changes (you don't update React every deploy), so browsers can cache it long-term. Only the `index` chunk changes with your application code. Without `manualChunks`, React would be bundled into the main chunk and the entire bundle would invalidate on every deploy.

</details>

---

### Q7

```tsx
// src/App.tsx
import { lazy, Suspense } from 'react';

const Settings = lazy(() => import('./pages/Settings'));
const Dashboard = lazy(() => import('./pages/Dashboard'));

export default function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/settings" element={<Settings />} />
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

#### ❓ How many JavaScript chunks are generated for this app?

<details>
<summary>✅ Answer</summary>

```txt
At minimum 3 chunks:
1. index-[hash].js       ← main entry (App.tsx + shared code + react-dom)
2. Settings-[hash].js    ← Settings page chunk (lazy import)
3. Dashboard-[hash].js   ← Dashboard page chunk (lazy import)
```

**Explanation:** Each dynamic `import()` call creates a separate chunk. `React.lazy(() => import('./pages/Settings'))` tells Rollup to split `Settings` into its own chunk. This chunk is only downloaded when the user navigates to `/settings`. If a user only uses `/dashboard`, they never download the Settings chunk. This is route-based code splitting — a fundamental optimization pattern.

</details>

---

### Q8

```ts
// src/utils.ts
export function used() {
  return 'I am used';
}

export function unused() {
  return 'I am never imported anywhere';
}

// src/main.ts
import { used } from './utils';
console.log(used());
```

#### ❓ Is `unused()` included in the production bundle?

<details>
<summary>✅ Answer</summary>

```txt
No — unused() is tree-shaken out of the bundle.
Only used() and its dependencies appear in the output.
```

**Explanation:** Vite's production build uses Rollup, which performs static analysis (tree shaking) to eliminate unused exports. Since `unused` is never imported anywhere in the application, Rollup detects it as dead code and removes it from the output bundle. This requires ES module syntax (`import`/`export`) — CommonJS (`require`/`module.exports`) is opaque to static analysis and cannot be tree-shaken the same way.

</details>

---

### Q9

```ts
// src/main.ts — development mode (npm run dev)
console.log(typeof module);
console.log(typeof require);
```

#### ❓ What are the values of `typeof module` and `typeof require` in a Vite dev server environment?

<details>
<summary>✅ Answer</summary>

```txt
"undefined"  ← typeof module
"undefined"  ← typeof require
```

**Explanation:** Vite's dev server uses native ES modules. In a native ESM environment, there is no `module` object and no `require` function — these are Node.js / CommonJS constructs. Vite serves files as ES modules to the browser, where CommonJS is not available. This is why packages that use `require()` at runtime (not just during build) need special handling. Some libraries check `typeof module !== 'undefined'` to detect CommonJS — this check returns `false` in Vite's browser environment.

</details>

---

### Q10

```bash
# package.json
{
  "scripts": {
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

```bash
npm run build
npm run preview
```

#### ❓ What does `vite preview` do and how is it different from `vite` (dev server)?

<details>
<summary>✅ Answer</summary>

```txt
vite preview: Serves the contents of the dist/ folder using a local HTTP server.
              Simulates production serving in development.
              Does NOT watch files or rebuild — static serving only.

vite (dev):   Runs the Vite dev server with HMR, native ESM serving,
              on-demand file transformation, and file watching.
```

**Explanation:** `vite preview` is used to verify the production build locally before deploying. It serves exactly what would be deployed to a CDN or static server. This is important for catching issues that only appear in the production build (different bundle splitting, minification edge cases, removed dead code). It runs on a different port (4173 by default) and has no HMR or file watching capabilities.

</details>

---

## 3. Configuration

### Q11

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  plugins: [react()],
});
```

```ts
// tsconfig.json — no paths configured
{
  "compilerOptions": {
    "baseUrl": "."
  }
}
```

```tsx
// src/components/Button.tsx
import { useAuth } from '@/hooks/useAuth';
```

#### ❓ The import resolves correctly in the browser. Does TypeScript show an error?

<details>
<summary>✅ Answer</summary>

```txt
Yes — TypeScript shows: "Cannot find module '@/hooks/useAuth' or its corresponding type declarations."
```

**Explanation:** Vite's alias configuration (`resolve.alias`) handles build-time module resolution. TypeScript's type checker uses a completely separate resolution mechanism — `tsconfig.json`'s `paths`. Both must be configured independently. The browser runs fine because Vite resolves the path. But the TypeScript server (used by your editor and `tsc`) doesn't know about Vite's aliases. Fix by adding `"paths": { "@/*": ["src/*"] }` to `tsconfig.json`.

</details>

---

### Q12

```ts
// vite.config.ts
export default defineConfig({
  plugins: [
    pluginA(),  // transforms JSX first
    pluginB(),  // reads JSX expecting raw source
  ],
});
```

#### ❓ In what order do Vite plugins execute during a transform?

<details>
<summary>✅ Answer</summary>

```txt
For 'transform' hooks: plugins execute in the ORDER they are listed in the array.
pluginA transforms first, pluginB receives pluginA's output.

Enforce ordering can override this:
- enforce: 'pre'  → runs before normal plugins
- enforce: 'post' → runs after normal plugins
```

**Explanation:** Unlike some build tools, Vite plugin order in the `plugins` array matters for transform hooks. Each plugin's `transform` receives the output of the previous plugin. If `pluginB` expects raw JSX but `pluginA` already transformed it to JavaScript, `pluginB` will receive the wrong input. Use `enforce: 'pre'` on a plugin that needs to run before others (like a preprocessor) and `enforce: 'post'` for plugins that should run last (like an analyzer).

</details>

---

### Q13

```ts
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
});
```

```ts
// src/service.ts
const res = await fetch('/api/users');
```

#### ❓ Where does the request actually go in development? Does the proxy apply in production builds?

<details>
<summary>✅ Answer</summary>

```txt
Development: fetch('/api/users') → Vite dev server → http://localhost:8080/api/users
The request URL appears as /api/users in the browser but hits the backend.

Production: The proxy does NOT exist. /api/users must be handled by:
- A real backend deployed at the same domain
- A reverse proxy (Nginx) configured in infrastructure
- An API gateway
```

**Explanation:** `server.proxy` only configures the Vite development HTTP server. When you run `vite build`, the proxy configuration is irrelevant — no Vite server runs in production. Your production deployment must have equivalent routing at the infrastructure level. Many teams pair Vite's dev proxy with an Nginx `location /api { proxy_pass ... }` block in production.

</details>

---

### Q14

```ts
// vite.config.ts
export default defineConfig({
  base: '/my-app/',
});
```

```tsx
// src/App.tsx
import logo from './assets/logo.png';

export default function App() {
  return <img src={logo} alt="Logo" />;
}
```

#### ❓ What is the `src` attribute on the `<img>` tag in the production build?

<details>
<summary>✅ Answer</summary>

```txt
/my-app/assets/logo-abc123.png
```

**Explanation:** The `base` option in Vite sets the public base path for all assets. In production, Vite prepends the base path to all asset URLs. This is necessary when deploying to a subdirectory (e.g., GitHub Pages at `https://username.github.io/my-app/`). Without `base: '/my-app/'`, assets would reference `/assets/logo.png` (root) instead of `/my-app/assets/logo.png`, causing 404s on subdirectory deployments.

</details>

---

### Q15

```ts
// vite.config.ts
import { defineConfig, loadEnv } from 'vite';

export default defineConfig(({ command, mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    define: {
      __APP_VERSION__: JSON.stringify(env.APP_VERSION),
    },
    plugins: [react()],
  };
});
```

```bash
# .env
APP_VERSION=2.1.0
```

```ts
// src/App.tsx
console.log(__APP_VERSION__);
```

#### ❓ What is logged, and why is `JSON.stringify` needed in `define`?

<details>
<summary>✅ Answer</summary>

```txt
"2.1.0"
(string, with quotes — because it's a JSON-stringified string)
```

**Explanation:** `define` performs a textual find-and-replace during build. Without `JSON.stringify`, `__APP_VERSION__` would be replaced with `2.1.0` (a bare token) which would be interpreted as a JavaScript identifier/expression — a syntax error. With `JSON.stringify('2.1.0')`, the replacement is `"2.1.0"` (a valid JavaScript string literal). `loadEnv(mode, cwd, '')` loads all env variables (the empty string prefix means no filtering — unlike `import.meta.env` which filters to `VITE_` prefix only).

</details>

---

## 4. Environment Variables

### Q16

```bash
# .env.production
VITE_API_URL=https://prod.api.com

# .env.development  
VITE_API_URL=http://localhost:3001

# .env.production.local
VITE_API_URL=https://staging.api.com
```

Developer runs `npm run build` (which uses `mode: 'production'`).

#### ❓ Which `VITE_API_URL` value is used in the production build?

<details>
<summary>✅ Answer</summary>

```txt
https://staging.api.com (from .env.production.local)
```

**Explanation:** Priority order for `mode: 'production'`:
1. `.env.production.local` ← highest priority → wins
2. `.env.local`
3. `.env.production`
4. `.env`

`.env.[mode].local` files always take priority over `.env.[mode]`. This is by design: `.local` files are typically git-ignored and contain developer-specific or environment-specific overrides. For a proper CI/CD setup, you typically don't commit `.env.production.local` and instead inject variables via CI environment.

</details>

---

### Q17

```ts
// src/config.ts
export const config = {
  apiUrl: import.meta.env.VITE_API_URL,
  mode: import.meta.env.MODE,
  isDev: import.meta.env.DEV,
};
```

This file is used both in the Vite app (browser) and imported into a Vite SSR plugin (Node.js server).

#### ❓ What is the value of `import.meta.env.SSR` in each context?

<details>
<summary>✅ Answer</summary>

```txt
Browser (client):   import.meta.env.SSR === false
Node.js (SSR):      import.meta.env.SSR === true
```

**Explanation:** `import.meta.env.SSR` is a boolean flag Vite sets to differentiate between client and server execution contexts. This allows you to write isomorphic code that behaves differently based on environment. For example, `if (!import.meta.env.SSR) { window.localStorage.setItem(...) }` safely accesses browser APIs only on the client. Framework integrations like Vite + React SSR, SvelteKit, and Nuxt use this flag internally.

</details>

---

### Q18

```ts
// src/App.tsx — Vite project
if (process.env.NODE_ENV === 'production') {
  initErrorTracking();
}
```

#### ❓ Does this code work in a Vite project? Will `process.env.NODE_ENV` be defined?

<details>
<summary>✅ Answer</summary>

```txt
Yes — process.env.NODE_ENV IS defined in Vite projects.
It equals 'production' during vite build and 'development' during vite.
```

**Explanation:** Vite explicitly replaces `process.env.NODE_ENV` (unlike other `process.env.*` variables). This is done for compatibility with many npm packages that use `process.env.NODE_ENV` to detect the environment. Vite replaces it at build time: `'production'` in production builds and `'development'` in dev. However, for your own app code, prefer `import.meta.env.PROD` / `import.meta.env.DEV` which are the idiomatic Vite way.

</details>

---

### Q19

```ts
// vite.config.ts
export default defineConfig({
  // No envPrefix configured
});

// .env
VITE_PUBLIC_KEY=pk_live_123
MY_CUSTOM_VITE_VAR=hello
```

#### ❓ Which variable is exposed to the browser?

<details>
<summary>✅ Answer</summary>

```txt
Only VITE_PUBLIC_KEY is exposed.
MY_CUSTOM_VITE_VAR is NOT exposed — it doesn't START with VITE_.
```

**Explanation:** Vite checks if the variable name **starts with** the prefix (`VITE_` by default). `MY_CUSTOM_VITE_VAR` contains `VITE_` but does not start with it — the prefix check is anchored to the beginning of the name. You can change the prefix using `envPrefix` in `vite.config.ts`:
```ts
envPrefix: 'APP_'  // now APP_* variables are exposed
```
Or disable the prefix requirement entirely (not recommended for security):
```ts
envPrefix: ''  // exposes ALL env variables (dangerous)
```

</details>

---

### Q20

```ts
// vite.config.ts
export default defineConfig({
  envDir: './config/env',
});
```

#### ❓ What does `envDir` change and where does Vite look for `.env` files by default?

<details>
<summary>✅ Answer</summary>

```txt
Default: Vite looks for .env files in the project root (same directory as vite.config.ts).

With envDir: './config/env': Vite looks for .env files in ./config/env/ instead.
```

**Explanation:** By default, Vite expects `.env`, `.env.local`, `.env.development`, etc. to be in the project root directory (where `vite.config.ts` lives). The `envDir` option lets you move env files to a custom location — useful for monorepos or when you want to separate config files from code. The path is relative to the project root (where `vite.config.ts` is).

</details>

---

## 5. Edge Cases

### Q21

```ts
// src/utils/format.ts
import _ from 'lodash';

export function formatDate(date: Date): string {
  return _.format(date, 'YYYY-MM-DD');
}
```

`lodash` is a CommonJS package. The project runs fine in development.

#### ❓ Why does lodash work in development despite being CommonJS?

<details>
<summary>✅ Answer</summary>

```txt
Vite's dependency pre-bundling converts lodash from CJS to ESM using esbuild.
The pre-bundled ESM version is served from .vite/deps/lodash.js.
```

**Explanation:** On first `vite` run, Vite's optimizer (esbuild) scans `node_modules` for CommonJS packages and converts them to ES modules. The result is cached in `.vite/deps/`. This happens automatically for all detected CJS packages. The browser receives an ESM-compatible version of lodash. If you need to force a package to be pre-bundled (e.g., it's not detected automatically), add it to `optimizeDeps.include` in your config.

</details>

---

### Q22

```ts
// src/App.tsx
import data from './data.json';

console.log(data.users.length);
```

```json
// src/data.json
{
  "users": [1, 2, 3]
}
```

#### ❓ Does Vite support importing JSON files directly? What does `data` contain?

<details>
<summary>✅ Answer</summary>

```txt
Yes — Vite natively supports JSON imports.
data is the parsed JavaScript object: { users: [1, 2, 3] }
console.log outputs: 3
```

**Explanation:** Vite transforms JSON files into ES module exports automatically — no plugins needed. The JSON is parsed and exported as the default export. In the production build, the JSON content is inlined into the bundle (not fetched separately). For large JSON files that should be fetched lazily, use `fetch('/data.json')` with the file in `public/` instead of importing it.

</details>

---

### Q23

```ts
// src/worker.ts
self.onmessage = ({ data }) => {
  const result = heavyComputation(data);
  self.postMessage(result);
};

// src/App.tsx
import MyWorker from './worker.ts?worker';

const worker = new MyWorker();
worker.postMessage({ input: 42 });
worker.onmessage = ({ data }) => console.log(data);
```

#### ❓ What does the `?worker` query suffix do in the import?

<details>
<summary>✅ Answer</summary>

```txt
?worker transforms the import to return a Web Worker constructor class.
new MyWorker() creates a Web Worker that runs worker.ts in a separate thread.
The worker.ts file is bundled into its own separate JavaScript file.
```

**Explanation:** Vite provides special import query suffixes for assets. `?worker` tells Vite to treat the file as a Web Worker and return a constructor class. The worker file is bundled separately (its own chunk) and runs in a background thread — off the main UI thread. Other query suffixes: `?url` (returns the URL), `?raw` (returns file content as string), `?inline` (returns data URI), `?worker&url` (returns worker URL instead of constructor).

</details>

---

### Q24

```ts
// src/App.tsx
const imagePath = './assets/logo.png';
import(imagePath);  // Dynamic import with variable path
```

#### ❓ Can Vite statically analyze this dynamic import? What happens at build time?

<details>
<summary>✅ Answer</summary>

```txt
Vite CANNOT statically analyze a dynamic import with a runtime variable path.
It may warn about this and cannot pre-build the asset into a deterministic chunk.
The import may fail at runtime if the path isn't resolvable.
```

**Explanation:** Rollup (Vite's production bundler) performs static analysis — it needs to know paths at build time to include assets in the bundle. A dynamic import with a fully variable path (`import(variable)`) is opaque to static analysis. Vite can analyze patterns like `import('./assets/' + name + '.png')` and include all matching files, but a bare variable like `imagePath` cannot be analyzed. For dynamic asset loading, place files in `public/` and use `fetch` or `<img src>` with the runtime URL.

</details>

---

### Q25

```ts
// src/App.tsx
export default function App() {
  if (import.meta.env.DEV) {
    console.log('Debug mode active');
  }
  return <div>Hello</div>;
}
```

#### ❓ What happens to the `if (import.meta.env.DEV) { ... }` block in the production build?

<details>
<summary>✅ Answer</summary>

```txt
The entire if block is removed from the production bundle (dead code elimination).

In production build, Vite replaces import.meta.env.DEV with false:
  if (false) {
    console.log('Debug mode active');
  }
Rollup/minifier then removes the dead code entirely.
```

**Explanation:** Vite statically replaces `import.meta.env.DEV` with `true` in development and `false` in production builds. After this replacement, the minifier (esbuild or terser) detects `if (false) { ... }` as unreachable code and removes it. This means debug-only code has zero cost in production — no bundle size increase, no runtime check. This pattern is called "compile-time dead code elimination" and is why `import.meta.env.DEV` checks are preferred over runtime `typeof process !== 'undefined'` guards.

</details>

---

## Topics Covered

| Category | Questions | Concepts Tested |
|---|---|---|
| Dev Server | Q1–Q5 | process.env vs import.meta.env, VITE_ prefix requirement, built-in env variables, static inlining, .env file priority |
| Build Output | Q6–Q10 | manualChunks, code splitting, tree shaking, typeof module/require, vite preview |
| Configuration | Q11–Q15 | tsconfig paths vs vite aliases, plugin order, proxy in dev only, base path, define with JSON.stringify |
| Environment Variables | Q16–Q20 | .env.production.local priority, SSR flag, process.env.NODE_ENV compatibility, prefix anchoring, envDir |
| Edge Cases | Q21–Q25 | CJS pre-bundling, JSON import, ?worker import query, dynamic import analysis, DEV dead code elimination |
