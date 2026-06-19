# Heap Snapshot Formatter

> **Layer:** Formatters · **Sources:** `src/formatters/HeapSnapshotFormatter.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Formats V8 heap-snapshot data into compact, CSV-style text and JSON: per-class size aggregates, lists of nodes/edges, retaining paths (the chain that keeps an object alive), and dominator chains. Byte sizes are rendered as KB for readability.

## Key files
- `HeapSnapshotFormatter.ts` — the formatter, the static formatting helpers, and the `isNodeLike`/`isEdgeLike` type guards.

## Key types & functions
- `HeapSnapshotFormatter` — instance class over `Record<string, AggregatedInfoWithId>` (class aggregates). Instance `toString()`/`toJSON()` render the aggregate table.
- `FormattedSnapshotEntry` — JSON shape per aggregate: `{className, id?, count, selfSize, retainedSize}` (sizes are KB strings).
- `HeapSnapshotFormatter.formatNodes(items)` — static; CSV for a node *or* edge list, header chosen by sniffing the first item.
- `HeapSnapshotFormatter.formatRetainingPaths(paths)` — static; recursive indented tree of retaining edges.
- `HeapSnapshotFormatter.formatDominators(chain)` — static; CSV of the dominator chain.
- `HeapSnapshotFormatter.sort(aggregates)` — static; returns `[key, AggregatedInfo]` entries sorted by `maxRet` desc (used by `McpResponse` before pagination).
- `isNodeLike` / `isEdgeLike` — duck-typing guards distinguishing `Node` vs `Edge` items.
- `#getSortedAggregates()` — sorts the instance's aggregates by `maxRet` (max retained size) descending.

## How it works
Unlike the per-item formatters, there is **no concise/detailed split** — instead there are several *table* renderers, each a flat CSV-ish block (cheap for the agent to scan), and one JSON variant for the aggregate table. All byte sizes go through `DevTools.I18n.ByteUtilities.formatBytesToKb`.

**Aggregate table** (instance `toString`), sorted by retained size desc:
```
id,name,count,selfSize,maxRetainedSize
<id>,<name>,<count>,<self KB>,<maxRet KB>
```
`toJSON()` emits the same rows as `FormattedSnapshotEntry[]` (`className`, `selfSize`, `retainedSize`). The stable id comes from `info[stableIdSymbol]` (`''` in text when absent).

**Nodes/edges** (`formatNodes`): header sniffed from `items[0]`:
- nodes → `nodeId,nodeName,type,distance,selfSize,retainedSize`
- edges → `name,type,nodeId,nodeName`

**Retaining paths** (`formatRetainingPaths`): depth-indented (2 spaces/level), each edge:
```
<- @<nodeId> <nodeName> via <edgeType> <edgeName> (distance: <distance>)
```

**Dominators** (`formatDominators`): `nodeId,nodeName,selfSize,retainedSize` rows.

## Relationships
- **Depends on:** `AggregatedInfoWithId` from [`src/HeapSnapshotManager.ts`](../06-performance-memory/); `stableIdSymbol` from `src/utils/id.js`; DevTools `HeapSnapshotModel` types and `I18n.ByteUtilities.formatBytesToKb` via `src/third_party/index.js`.
- **Used by:** [`McpResponse`](../03-context-state/) — calls `HeapSnapshotFormatter.sort` then paginates, builds `new HeapSnapshotFormatter(paginatedRecord)` for the aggregate table (`toJSON()` → `structuredContent.heapSnapshotData`, `toString()`/TOON for text), and the static `formatDominators` (and the node/edge/retaining-path helpers) for the corresponding sections.

## Gotchas & non-obvious details
- Sorting is always by **retained** size (`maxRet`), not self size — the most "expensive to free" classes surface first.
- All sizes are pre-formatted KB **strings**, not raw bytes, in both text and JSON.
- `formatNodes` picks its header from the first item only; a mixed node/edge list (shouldn't happen) would mislabel rows.
- The stable id is a `Symbol`-keyed property (`stableIdSymbol`), not an enumerable field, so it survives only through these formatters' explicit reads.

## Update triggers
- CSV column sets/order or the retaining-path line format change.
- Sort key (`maxRet`) or KB formatting changes.
- `FormattedSnapshotEntry` shape changes, or new heap views (beyond nodes/edges/paths/dominators/aggregates) are added.
