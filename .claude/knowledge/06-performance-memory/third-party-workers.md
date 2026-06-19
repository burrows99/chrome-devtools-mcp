# Third-Party Boundary & Workers (`src/third_party/*`)

> **Layer:** Performance & memory engine · **Sources:** `src/third_party/index.ts`, `src/third_party/devtools-formatter-worker.ts`, `src/third_party/devtools-heap-snapshot-worker.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`src/third_party/` is the single, controlled boundary between first-party server code and the vendored Chrome DevTools frontend + Lighthouse bundles. `index.ts` re-exports everything the rest of the codebase is allowed to import; the two `*-worker.ts` files are thin entrypoints that load DevTools worker bundles into a worker thread.

## Key files
- `src/third_party/index.ts` — the re-export hub: `DevTools`, Lighthouse functions, and shared third-party deps.
- `src/third_party/devtools-formatter-worker.ts` — entrypoint for the DevTools formatter worker.
- `src/third_party/devtools-heap-snapshot-worker.ts` — entrypoint for the DevTools heap-snapshot worker.
- `src/third_party/lighthouse-devtools-mcp-bundle.js` — pre-built (~8 MB), minified Lighthouse bundle. **Not first-party; not edited by hand.**
- `src/third_party/LIGHTHOUSE_MCP_BUNDLE_THIRD_PARTY_NOTICES` — license notices for the bundle.

## Key exports (from `index.ts`)
- **`DevTools`** — `export * as DevTools from '../../node_modules/chrome-devtools-frontend/mcp/mcp.js'`. The entire vendored DevTools MCP surface: `TraceEngine`, `HeapSnapshotModel`, `PerformanceTraceFormatter`, `PerformanceInsightFormatter`, `AgentFocus`, `CrUXManager`, `Common.Settings`, `I18n`, `Formatter`, etc.
- **`snapshot` / `navigation` / `generateReport`** — the three Lighthouse functions, imported from `./lighthouse-devtools-mcp-bundle.js` and re-typed against the `lighthouse` package's `Flags` / `Result` / `RunnerResult` types.
- **Shared deps** re-exported for the whole project: `McpServer` + transports/types from `@modelcontextprotocol/sdk`, `zod`, `ajv`, `yargs`/`hideBin`, `semver`, `debug`, `puppeteer-core` (incl. `Locator`, `PredefinedNetworkConditions`, `KnownDevices`, `PipeTransport`), `@puppeteer/browsers` helpers, `@toon-format/toon` `toonEncode`.
- **Polyfill side-effects** at the top: `urlpattern-polyfill`, `Promise.withResolvers`, `Set.union`, iterator-helpers (core-js). These are required for the vendored bundle to run under Node.

## How it works
**The `index.ts` boundary (first-party).** The project's lint rules restrict direct imports of `chrome-devtools-frontend` / Lighthouse internals (note the `eslint-disable no-restricted-imports` in the worker files). All consumers import from `'../third_party/index.js'` instead, so there is exactly one place that knows the vendored layout. The Lighthouse functions are cast to clean signatures (e.g. `snapshot: (page, {flags}) => Promise<RunnerResult>`) because they come out of a minified bundle untyped.

**The two worker entrypoints (first-party, one line each).** Each file is a single side-effecting import of a DevTools worker bundle:
```ts
// devtools-heap-snapshot-worker.ts
import '../../node_modules/chrome-devtools-frontend/front_end/entrypoints/heap_snapshot_worker/heap_snapshot_worker-entrypoint.js';
```
```ts
// devtools-formatter-worker.ts
import '../../node_modules/chrome-devtools-frontend/front_end/entrypoints/formatter_worker/formatter_worker-entrypoint.js';
```
They exist as separate compiled `.js` files so they can be loaded as the *worker thread's* entry module. The vendored DevTools workers expect to run in a Worker context; these wrappers are what the worker proxy actually executes.

**Who loads them (first-party callers, outside this layer's files):**
- `HeapSnapshotManager.#loadSnapshot` resolves `./third_party/devtools-heap-snapshot-worker.js` via `import.meta.resolve(...)` and passes it to `HeapSnapshotWorkerProxy` (see [heap-snapshot-manager.md](./heap-snapshot-manager.md)).
- `src/devtools/DevtoolsUtils.ts` resolves `../third_party/devtools-formatter-worker.js` and passes it to `DevTools.Formatter.FormatterWorkerPool.instance({forceNew, entrypointURL})`.

**How the Lighthouse bundle is produced (do NOT document line-by-line).** `scripts/update-lighthouse.ts` runs `yarn build-devtools-mcp` in a sibling `../lighthouse` checkout, then copies `dist/lighthouse-devtools-mcp-bundle.js` and its notices into `src/third_party/`. The DevTools frontend itself comes straight from the pinned `chrome-devtools-frontend` npm dependency. See [Build tooling](../09-build-tooling/lighthouse-vendoring.md).

## Relationships
- **Depends on:** pinned `chrome-devtools-frontend@1.0.1646286` and `lighthouse@13.4.0` (versions in `package.json`); [Build tooling](../09-build-tooling/lighthouse-vendoring.md).
- **Used by:** essentially everything — [trace processing](./trace-processing.md), [HeapSnapshotManager](./heap-snapshot-manager.md), [lighthouse tool](../02-tools/groups/), all tools needing `zod`/`puppeteer`/MCP types.

## Gotchas & non-obvious details
- **`lighthouse-devtools-mcp-bundle.js` is generated, minified, and huge** — never hand-edit it; regenerate via `scripts/update-lighthouse.ts`. The build separately strips `build/node_modules` and appends license notices (see the `bundle` script in `package.json`).
- **Version pinning matters:** the worker entrypoints and `DevTools` exports must match the installed `chrome-devtools-frontend` version exactly; a bump can move/rename entrypoint paths.
- The worker `.ts` files are the only place `no-restricted-imports` is intentionally disabled.
- `post-build.ts` writes several **mock** DevTools modules (i18n locales, `codemirror.next`, `core/root/Runtime.js`) into `build/` so the vendored bundle runs headless under Node — another sign the vendored code assumes a browser/DevTools runtime.
- `mcp/mcp.js` is the DevTools-side aggregated entrypoint; `DevTools` is `export *`, so its surface is whatever that file exposes for the installed version.

## Update triggers
- `scripts/update-lighthouse.ts` is run, or `lighthouse` / `chrome-devtools-frontend` versions in `package.json` change.
- DevTools moves the `heap_snapshot_worker` / `formatter_worker` entrypoint paths, or changes `mcp/mcp.js`'s exports.
- New polyfills are required by the vendored bundle.
