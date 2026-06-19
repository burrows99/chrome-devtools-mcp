# Build & Bundle

> **Layer:** Build & tooling · **Sources:** `package.json`, `tsconfig.json`, `scripts/post-build.ts`, `rollup.config.mjs`, `puppeteer.config.cjs`, `scripts/prepare.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Compile TypeScript to `build/` (development run target) and, for publishing, bundle all `node_modules` dependencies into a small set of self-contained ESM files under `src/third_party` so the npm tarball is dependency-free for runtime.

## Key files
- `tsconfig.json` — compiler options + the long `include` list of `chrome-devtools-frontend` source dirs
- `scripts/post-build.ts` — writes DevTools shims/mocks into `build/` after `tsc`
- `rollup.config.mjs` — three Rollup bundles + license harvesting
- `puppeteer.config.cjs` — only downloads `chrome`, skips headless-shell/firefox
- `scripts/prepare.ts` — `prepare` hook; patches `node_modules` after install

## How it works
**`npm run build`** = `tsc && node scripts/post-build.ts`.
- `tsc` emits to `build/` (`outDir: ./build`, `rootDir: .`), so output lands at `build/src/...` and `build/tests/...`. It also compiles selected `chrome-devtools-frontend/front_end/...` dirs listed in `tsconfig.json`.
- `post-build.ts` writes JS mocks into `build/` that DevTools modules import but that aren't needed at runtime: an i18n `locales.js`, a `codemirror.next.js` stub, a `core/root/Runtime.js` mock (experiment flags etc.), copies a few CodeMirror `.mjs` files that `tsc` skips, and `cpSync`s the issues_manager `descriptions/` into `build/src/third_party/issue-descriptions`.

**`npm run bundle`** chains:
```
npm run clean                      # rm -rf build
&& npm run build                   # tsc + post-build
&& rollup -c rollup.config.mjs     # inline node_modules into 3 third_party bundles
&& rm -rf build/node_modules       # drop the now-inlined deps from the tarball
&& node scripts/append-lighthouse-notices.ts
```
Rollup re-writes `build/src/third_party/index.js`, `devtools-formatter-worker.js`, and `devtools-heap-snapshot-worker.js` in place as `format: 'esm'` with `inlineDynamicImports`. Plugins: `cleanup` (strips comments but keeps `/Copyright/i`), `license` (allow-lists licenses, fails on unlicensed/violation, writes `THIRD_PARTY_NOTICES`), `commonjs`, `json`, `nodeResolve`. A custom `listBundledDeps` plugin writes `bundled-packages.json`. `NODE_ENV=production` disables sourcemaps. See [lighthouse-vendoring](lighthouse-vendoring.md) for the notices step.

**`npm run prepare`** (auto-run on `npm install`) runs `scripts/prepare.ts`, which deletes a few problematic files from `node_modules/chrome-devtools-frontend` and strips a conflicting `declare global { interface HTMLElementEventMap }` block from `@paulirish/trace_engine` to avoid TS2717.

## Relationships
- **Depends on:** `src/third_party/index.ts` barrel (re-exports the bundled packages), `node_modules` layout, `package.json#devDependencies`.
- **Produces / affects:** `build/` (run via `npm start`), the published bundle consumed by [Performance & memory](../06-performance-memory/README.md), and `package.json#files` (`build/src`, `LICENSE`, `skills`).

## Gotchas & non-obvious details
- The package ships the Rollup bundle, NOT `build/node_modules` — a direct `import 'somePkg'` in `src/` would break after publish. The `@local/no-direct-third-party-imports` ESLint rule guards this (see [config-files](config-files.md)).
- `prepare` mutates installed `node_modules` — re-runs on every fresh install.
- Build requires Node 22+ even though runtime supports Node 20 (`run-tests.yml` builds on 22).

## Update triggers
- A new DevTools module needs a shim → edit `post-build.ts`.
- A new third_party bundle entrypoint is added → edit `rollup.config.mjs`.
- `chrome-devtools-frontend` / `@paulirish/trace_engine` upgrades break `prepare.ts` patches.
