# Performance & Memory Engine

> **Layer:** Performance & memory engine · **Sources:** `src/trace-processing/parse.ts`, `src/HeapSnapshotManager.ts`, `src/third_party/index.ts`, `src/third_party/devtools-formatter-worker.ts`, `src/third_party/devtools-heap-snapshot-worker.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
This layer is the thin first-party glue that drives Chrome's **vendored** trace engine and heap-snapshot machinery. It parses recorded performance-trace buffers into structured insights, loads `.heapsnapshot` files into a worker-backed model and exposes aggregates/dominators/retainers/edges, and re-exports the Lighthouse and `chrome-devtools-frontend` bundles for the rest of the server.

## Key files
- `src/trace-processing/parse.ts` — parse a raw trace buffer into a `ParsedTrace` + insights; format summaries/insights for the LLM.
- `src/HeapSnapshotManager.ts` — load and cache heap snapshots; expose aggregate/dominator/retainer/edge queries via a worker proxy.
- `src/third_party/index.ts` — the single import boundary that re-exports the vendored DevTools frontend, Lighthouse functions, and shared third-party deps.
- `src/third_party/devtools-formatter-worker.ts` — one-line worker entrypoint re-exporting DevTools' `formatter_worker`.
- `src/third_party/devtools-heap-snapshot-worker.ts` — one-line worker entrypoint re-exporting DevTools' `heap_snapshot_worker`.

## First-party vs vendored — the critical boundary
Almost all of the *heavy lifting* lives in the vendored bundle, NOT in this layer:

- **`node_modules/chrome-devtools-frontend`** (version pinned in `package.json` → `chrome-devtools-frontend: 1.0.1646286`) provides the entire `TraceEngine`, `HeapSnapshotModel`, formatters (`PerformanceTraceFormatter`, `PerformanceInsightFormatter`), `CrUXManager`, `I18n`, and the two worker entrypoints. It is re-exported wholesale as `DevTools` from `src/third_party/index.ts` (`export * as DevTools from '../../node_modules/chrome-devtools-frontend/mcp/mcp.js'`).
- **`src/third_party/lighthouse-devtools-mcp-bundle.js`** is an ~8 MB pre-built, minified Lighthouse bundle (regenerated, not hand-edited). It is produced by building Lighthouse's `build-devtools-mcp` target in a sibling `../lighthouse` checkout and copied in by `scripts/update-lighthouse.ts`. Do NOT try to document its internals line-by-line. Lighthouse version is pinned in `package.json` → `lighthouse: 13.4.0`.

The first-party code in this layer is intentionally tiny: it instantiates vendored models, feeds them buffers/files, and hands the results to the formatter layer. Treat the vendored APIs (`DevTools.*`) as an external SDK.

## How the pieces fit
**Trace flow:** `performance_start_trace` / `performance_stop_trace` (tools) record a CDP trace via Puppeteer → the raw buffer is handed to `parseRawTraceBuffer` → the vendored `TraceModel.Model` parses events into a `ParsedTrace` with `insights` → `getTraceSummary` / `getInsightOutput` format the data via vendored `PerformanceTraceFormatter` / `PerformanceInsightFormatter` for `performance_analyze_insight`.

**Heap flow:** `take_heapsnapshot` (tool) captures a `.heapsnapshot` file via CDP → `HeapSnapshotManager` streams the file into a `HeapSnapshotWorkerProxy` (running `devtools-heap-snapshot-worker.js`) → the manager queries the resulting `HeapSnapshotProxy` for aggregates, statistics, nodes-by-class, retainers, retaining paths, dominators, and edges → `HeapSnapshotFormatter` renders CSV/JSON for the memory tools.

**Worker boundary:** the two `.ts` files in `src/third_party/` are *entrypoints* — each is a single side-effecting import of a DevTools worker bundle. They are compiled by `tsc` to `.js` and loaded at runtime by worker proxies that resolve them via `import.meta.resolve(...)` (see `HeapSnapshotManager.#loadSnapshot` and `DevtoolsUtils.ts` `FormatterWorkerPool`).

## File index
- [`trace-processing.md`](./trace-processing.md) — `parse.ts`: inputs, outputs, vendored APIs called.
- [`heap-snapshot-manager.md`](./heap-snapshot-manager.md) — `HeapSnapshotManager`: API, caching, worker loading.
- [`third-party-workers.md`](./third-party-workers.md) — `src/third_party/*`: the `index.ts` boundary and the two worker entrypoints.

## Relationships
- **Depends on:** vendored `chrome-devtools-frontend` & `lighthouse` bundles; [Build tooling](../09-build-tooling/lighthouse-vendoring.md) for how the bundle is produced.
- **Used by:** [performance tool](../02-tools/groups/performance.md), [memory tool](../02-tools/groups/memory.md), [HeapSnapshotFormatter](../05-formatters/heap-snapshot-formatter.md).

## Update triggers
- `scripts/update-lighthouse.ts` is run or the `lighthouse` / `chrome-devtools-frontend` versions in `package.json` change.
- The vendored trace insight set changes (new/removed `InsightModels` keys).
- The `HeapSnapshotModel` proxy API (aggregates/dominators/retainers/edges) changes.
