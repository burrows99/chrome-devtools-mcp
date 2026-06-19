# Memory tools

> **Layer:** Tool system · **Sources:** `src/tools/memory.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Captures and inspects V8 heap snapshots to analyze JS object memory distribution and debug leaks. One tool captures a `.heapsnapshot`; the rest load a previously saved snapshot and query stats, aggregates, class instances, retainers, retaining paths, edges, and dominators via the `HeapSnapshotManager`.

## Key files
- `src/tools/memory.ts` — defines `take_heapsnapshot` plus eight `*_heapsnapshot*` read/query tools.

## Tools exposed
All use `ToolCategory.MEMORY`. Every tool except `take_heapsnapshot` is gated on the `memoryDebugging` condition flag.
- `take_heapsnapshot` — page-scoped. Params: `filePath` (string). Captures via `page.pptrPage.captureHeapSnapshot({path})`; `ensureExtension(filePath, '.heapsnapshot')` enforces the extension. `readOnlyHint: false`, `blockedByDialog: true`.
- `get_heapsnapshot_summary` — Params: `filePath`. Calls `context.getHeapSnapshotStats` + `getHeapSnapshotStaticData`; `response.setHeapSnapshotStats(stats, staticData)`.
- `get_heapsnapshot_details` — Params: `filePath`, `pageIdx?`, `pageSize?`. Calls `context.getHeapSnapshotAggregates`; `response.setHeapSnapshotAggregates(aggregates, {pageIdx, pageSize})`. Aggregates are paginated.
- `get_heapsnapshot_class_nodes` — Params: `filePath`, `id` (class id from details), `pageIdx?`, `pageSize?`. Calls `context.getHeapSnapshotNodesById`; `response.setHeapSnapshotNodes`.
- `get_heapsnapshot_retainers` — Params: `filePath`, `nodeId`, `pageIdx?`, `pageSize?`. Calls `context.getHeapSnapshotRetainers`; `response.setHeapSnapshotNodes`.
- `get_heapsnapshot_retaining_paths` — Params: `filePath`, `nodeId`, `maxDepth?`, `maxNodes?`, `maxSiblings?`. Calls `context.getHeapSnapshotRetainingPaths`; `response.setHeapSnapshotRetainingPaths`.
- `get_heapsnapshot_edges` — Params: `filePath`, `nodeId`, `pageIdx?`, `pageSize?`. Calls `context.getHeapSnapshotEdges`; `response.setHeapSnapshotNodes`.
- `get_heapsnapshot_dominators` — Params: `filePath`, `nodeId`. Calls `context.getHeapSnapshotDominators`; `response.setHeapSnapshotDominators`.
- `close_heapsnapshot` — Params: `filePath`. Calls `context.closeHeapSnapshot`; throws if the snapshot was not loaded. `readOnlyHint: false`.

All query tools have `readOnlyHint: true` except `take_heapsnapshot` and `close_heapsnapshot`. All declare `verifyFilesSchema: ['filePath']` so the path is validated by the handler before execution.

## How it works
- The query tools are thin wrappers: they forward `filePath`/`nodeId`/pagination params to matching `context.getHeapSnapshot*` methods, which delegate to the `HeapSnapshotManager`.
- The manager lazily loads a snapshot into a DevTools `HeapSnapshotWorkerProxy` (streamed in 1 MB chunks via `createReadStream`), caches it by absolute path, and assigns stable numeric ids to class keys (`getOrCreateIdForClassKey`) so `id` from details maps back to a class name in `get_heapsnapshot_class_nodes`.
- `nodeId`-based queries first resolve `snapshot.nodeIndexForId(nodeId)` and throw `Node with ID ${nodeId} not found` when absent.
- `close_heapsnapshot` -> `manager.dispose(filePath)` disposes the worker and drops the cache entry, returning `false` (-> tool throws) if nothing was loaded.

## Relationships
- **Depends on:** [`src/HeapSnapshotManager.ts`](../../../../src/HeapSnapshotManager.ts) (all heap queries route through it via `Context`), `ensureExtension` in `src/utils/files.ts`, and the DevTools `HeapSnapshotModel` types for the response setters.
- **Used by:** the MCP `ToolHandler`; response setters (`setHeapSnapshot*`) are defined on the `Response` interface in `src/tools/ToolDefinition.ts`.

## Gotchas & non-obvious details
- Only `take_heapsnapshot` is always available; the eight inspection tools require the experimental `memoryDebugging` flag (condition), so they are hidden unless enabled.
- The `id` accepted by `get_heapsnapshot_class_nodes` is a server-assigned stable class id obtained from `get_heapsnapshot_details`, NOT a node id; `nodeId` (used by retainers/paths/edges/dominators) is the V8 node id.
- Snapshots are cached in-process and never auto-freed — call `close_heapsnapshot` to release worker memory.
- `take_heapsnapshot` is `blockedByDialog: true`; the read tools are not.

## Update triggers
- A new `*_heapsnapshot*` tool is added or a param/pagination shape changes in `src/tools/memory.ts`.
- The `Context.getHeapSnapshot*` surface or `HeapSnapshotManager` method names change.
- The `memoryDebugging` condition flag is renamed or removed.
