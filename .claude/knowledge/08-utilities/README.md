# Utilities & Shared Types

> **Layer:** Utilities & shared types · **Sources:** `src/utils/files.ts`, `src/utils/id.ts`, `src/utils/keyboard.ts`, `src/utils/pagination.ts`, `src/utils/string.ts`, `src/utils/types.ts`, `src/types.ts`, `src/utils/check-for-updates.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Small, dependency-light helper modules and the top-level shared TypeScript interfaces that the rest of the server reuses. These are pure leaf utilities (filesystem path helpers, ID generation, keyboard parsing, list pagination, string casing) plus the cross-cutting type definitions (`EmulationSettings`, `TextSnapshotNode`, `PaginationOptions`, etc.).

## Key files
- `src/utils/files.ts` — temp-file paths, extension normalization, canonical/symlink-resolved path resolution.
- `src/utils/id.ts` — monotonic ID generator + `stableIdSymbol` scheme that underpins resource UID references (snapshots, pages, workers, heap snapshots).
- `src/utils/keyboard.ts` — parses key combos like `Shift+Enter` into a Puppeteer `KeyInput` tuple; validates against an allowlist.
- `src/utils/pagination.ts` — generic `paginate()` over any array, returns page slice + navigation metadata.
- `src/utils/string.ts` — Unicode-aware `toSnakeCase()`.
- `src/utils/types.ts` — `PaginationOptions` (the input shape consumed by `paginate()`).
- `src/types.ts` — top-level shared interfaces: `ExtensionServiceWorker`, `TextSnapshotNode`, `GeolocationOptions`, `EmulationSettings`, `Logger`.
- `src/utils/check-for-updates.ts` — entrypoint-only helper that checks npm for a newer version and warns; cross-linked here, documented briefly.

## Documents
- [files-and-pagination.md](files-and-pagination.md) — `files.ts`, `pagination.ts`
- [id-string-keyboard.md](id-string-keyboard.md) — `id.ts`, `string.ts`, `keyboard.ts`
- [shared-types.md](shared-types.md) — `src/types.ts`, `src/utils/types.ts`, plus a note on `check-for-updates.ts`

## How it works
Each module is a handful of pure functions or type declarations with no internal state shared between them (the only stateful piece is the per-call closure inside `createIdGenerator()` and the module-level `isChecking` flag in `check-for-updates.ts`). They depend only on Node builtins and the project's `third_party` re-export barrel (for `KeyInput` and `semver`). The `id.ts` scheme is the most load-bearing: `stableIdSymbol` is attached to collected resources so the server can hand the model short numeric UIDs and later resolve them back to live objects — this is the foundation for accessibility-snapshot element references and similar resource handles.

## Relationships
- **Depends on:** Node builtins (`fs`, `os`, `path`, `child_process`, `process`); `src/third_party/index.js` (re-exports `KeyInput`, `Viewport`, `Target`, `SerializedAXNode`, `semver`); `src/version.js` (in `check-for-updates.ts`).
- **Used by:** the context/state layer (`McpContext`, `PageCollector`, `ServiceWorkerCollector`, `HeapSnapshotManager`), the response/formatter layer (`McpResponse`, `HeapSnapshotFormatter`), the tools layer (`tools/input.ts`, `tools/memory.ts`, `tools/screencast.ts`), telemetry (`telemetry/flagUtils.ts`), and the CLI entrypoints (`bin/*`).

## Gotchas & non-obvious details
- These utilities have almost no tests of their own here; correctness is exercised indirectly by their consumers. Treat them as stable primitives.
- `src/utils/types.ts` and `src/types.ts` are two different files with overlapping naming — keep them straight (see [shared-types.md](shared-types.md)).

## Update triggers
- A new module is added under `src/utils/`.
- The UID/`stableIdSymbol` scheme changes (signature, reset behavior, or who attaches it).
- A shared interface in `src/types.ts` or `src/utils/types.ts` gains/loses fields.
