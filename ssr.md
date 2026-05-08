# ESM Components for SSR Consumption in Next.js

## Context

The current system is a microfrontend delivery platform: components are built by `cds-cli` into ESM bundles, deployed to CDN, and loaded at runtime by `cds-loader` in the browser. The loader already has an explicit SSR bailout — `isInBrowser()` returns null/LoadingComponent on the server — so **SSR is intentionally skipped today**.

The question is: what would it take to actually render these components on the server?

---

## The Core Tension

The CDN delivery model is fundamentally browser-first. Components are fetched async from a CDN URL at runtime. SSR needs code **synchronously available** at render time, before the response is sent. These two things are in direct conflict.

There are three different things the user might mean by "SSR in Next.js", with very different implications:

---

## Option A — Keep CDN Delivery, Better SSR Shell (Low Effort)

**What it is:** Don't change the delivery model. Instead of rendering `null` on SSR, render a richer shell — a skeleton or a server-fetched static snapshot.

**What changes:**
- `cds-loader` renders a real skeleton on SSR instead of `null`
- No changes to `cds-cli` builds at all
- Maybe add React Suspense streaming support so the client hydrates incrementally

**Dependency handling:** Not a problem — components still load in the browser only.

**Bundling:** Nothing changes.

**Limitations:** Not true SSR. Search engines, social crawlers, and initial paint don't get component content.

---

## Option B — Dual Artifacts: CDN (browser) + npm Package (SSR)

**What it is:** `cds-cli` produces two outputs per component:
1. **Current ESM artifact** for CDN (browser runtime loading, unchanged)
2. **New SSR bundle** — an npm-publishable package for server-side import

The consuming Next.js app does `import Component1 from '@venture/component1-ssr'` on the server and uses the CDN loader on the client. You'd pair these with a `use client` / `use server` split in Next.js 13+ App Router, or a conditional import in Pages Router.

**What changes in `cds-cli`:**
- Add a new build target (call it `ssr` or `node`) that outputs a CJS or ESM module suitable for Node.js
- Key differences from the browser ESM build:
  - **No browser globals** (window, document) — these would need guards or stubs
  - **Externalize React differently** — instead of CDN fallback modules, externalize as Node CJS (so consuming app provides them from `node_modules`)
  - **Bundle or externalize `@ux/*` deps** — either bundle them in (bigger artifact) or require them in consuming app's `node_modules`
  - **CSS extraction** — Linaria/CSS-in-JS needs server-side extraction; `@linaria/shaker` handles this but needs wiring

**Dependency handling:** This is the crux. Two strategies:
- **Bundle everything** (self-contained): Zero peer deps, but large; may cause React version conflicts if component bundles its own React
- **Externalize React + framework deps**: Smaller, but consuming app must `npm install` the same versions — tight coupling, but this is how most component libraries work

**Bundling:** Yes, you'd bundle with a different webpack config (or rollup/tsup would be cleaner for this). The existing Rollup and esbuild engines in `cds-cli` are close to what you'd want — they're simpler than the ESM engine.

**What changes in `cds-loader`:**
- `ComponentLoader` needs a conditional: if server and SSR bundle available, render it directly
- Or: App Router pattern — a server component wrapper that imports the SSR bundle, with `use client` on the interactive parts

**Complexity:** Medium. The build change is mechanical; the harder part is the CSS pipeline and ensuring no browser globals leak.

---

## Option C — True Isomorphic CDN Loading (High Effort, Experimental)

**What it is:** Fetch and execute component code on the server the same way a browser would, using dynamic `import()` or `vm.runInContext`. Node 18+ supports ESM dynamic import of URLs.

```js
// This actually works in Node 18+:
const mod = await import('https://cdn.example.com/component.es.js')
```

**What changes:** Mostly in `cds-loader` — detect SSR, do an async CDN fetch + dynamic import, await it before render.

**Dependency handling:** This is where it breaks down. The component's externalized deps (React, @ux/*) are referenced as bare specifiers or CDN URLs in the import map. Node doesn't have import map support natively. You'd need:
- An experimental `--import` hook that implements import maps for Node
- Or bundle all deps into the component (defeats the externalization model)
- Or use a tool like `es-module-shims` polyfill for Node (fragile)

**Bundling:** You'd need self-contained bundles (no externals) for the CDN artifact to work in Node, OR a separate Node-targeted artifact.

**Limitations:** Complexity is high, Node ESM URL loading has security/caching implications, and the import map gap is a real blocker without bundling deps in.

---

## Practical Recommendation

**Option B (dual artifact)** is the pragmatic path if true SSR content matters. Here's the minimal scope:

1. **New `cds-cli` build target**: Add an `--ssr` or `--target node` flag that runs a simplified rollup/esbuild build with:
   - `react` and `react-dom` as externals (peerDeps)
   - `@ux/*` packages either bundled or externalized (user choice)
   - CSS extracted to a `.css` file via `@linaria/rollup`
   - Output: `dist/ssr.js` (CJS) or `dist/ssr.mjs` (ESM)

2. **New `package.json` exports field** in built component:
   ```json
   {
     "exports": {
       ".": {
         "node": "./dist/ssr.js",
         "default": "./dist/index.es.js"
       }
     }
   }
   ```

3. **`cds-loader` update**: In Next.js App Router, expose a server component path. In Pages Router, use `next/dynamic` with `{ ssr: true }` and the npm import.

**The biggest unknown** is `@ux/*` design system deps — if those aren't available as SSR-safe packages themselves, bundling them in is the safest path even if it increases bundle size.

---

## Key Files

- `packages/cds-cli/src/build/esm/` — Current ESM pipeline (4-phase webpack)
- `packages/cds-cli/src/build/rollup/` — Rollup engine (closer starting point for SSR build)
- `packages/cds-cli/src/build/esbuild/` — esbuild engine (simplest, good for SSR output)
- `packages/cds-cli/src/build/index.js` — Build dispatcher (where new `ssr` target would be added)
- `packages/cds-loader/src/component-loader.tsx` — Contains `isInBrowser()` bailout
- `packages/cds-loader/src/cds-utils.ts` — `isInBrowser()` definition
- `packages/samples/basic/component1/` — Reference component to test against

---

## Questions to Resolve Before Implementing

1. **Next.js version target**: Pages Router (12/13) vs App Router (13+/14+)? App Router's server/client split is cleaner for this.
2. **`@ux/*` packages**: Are they SSR-safe? Do they have browser-only code? This determines bundle-or-externalize.
3. **CSS**: Is Linaria the only CSS mechanism, or are there global stylesheets? SSR CSS extraction is non-trivial.
4. **Hydration strategy**: Do components need to be interactive after SSR (full hydration) or is static rendering enough?
