# Shared Types (and update-check helper)

> **Layer:** Utilities & shared types · **Sources:** `src/types.ts`, `src/utils/types.ts`, `src/utils/check-for-updates.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The top-level shared interfaces reused across the server (`src/types.ts`), the pagination input shape (`src/utils/types.ts`), and a brief note on the entrypoint-only update-check helper (`src/utils/check-for-updates.ts`).

## Key files
- `src/types.ts` — cross-cutting domain interfaces.
- `src/utils/types.ts` — `PaginationOptions` only.
- `src/utils/check-for-updates.ts` — npm version-check side-helper, called from the CLI entrypoints.

## Key functions / types
From `src/types.ts`:
- `ExtensionServiceWorker` — `{ url: string; target: Target; id: string }`. A discovered extension service worker.
- `TextSnapshotNode extends SerializedAXNode` — accessibility node enriched with `id: string`, optional `backendNodeId?: number`, optional `loaderId?: string`, and `children: TextSnapshotNode[]`. The tree node type for the text accessibility snapshot.
- `GeolocationOptions` — `{ latitude: number; longitude: number }`.
- `EmulationSettings` — all-optional emulation knobs: `networkConditions?`, `cpuThrottlingRate?`, `geolocation?`, `userAgent?`, `colorScheme?: 'dark'|'light'`, `viewport?: Viewport`, `extraHttpHeaders?: Record<string,string>`.
- `Logger = ((...args: unknown[]) => void) | undefined` — optional logging callback threaded through the server.

From `src/utils/types.ts`:
- `PaginationOptions` — `{ pageSize?: number; pageIdx?: number }`. The input consumed by `paginate()` (see [files-and-pagination.md](files-and-pagination.md)).

From `src/utils/check-for-updates.ts`:
- `checkForUpdates(message: string): Promise<void>` — compares `VERSION` against a cached latest version and warns if behind; throttled to once/24h.
- `resetUpdateCheckFlagForTesting()` — `@internal`, resets the module-level `isChecking` guard for tests.

## How it works
`src/types.ts` is pure type declarations re-exporting/extending types from `src/third_party/index.js` (`SerializedAXNode`, `Viewport`, `Target`). No runtime code.

`check-for-updates.ts`: guarded by a module-level `isChecking` flag and the `CHROME_DEVTOOLS_MCP_NO_UPDATE_CHECKS` env var. Reads a cached `~/.cache/chrome-devtools-mcp/latest.json`; if the cached version is greater than the running `VERSION` (`semver.lt`) it prints an "Update available" warning. It bumps the cache file's mtime immediately (to stop concurrent subprocesses) and, at most once per 24h, spawns a **detached, unref'd** child process (`bin/check-latest-version.js`) to fetch the newest version into the cache. Every filesystem/spawn step is wrapped in try/catch that swallows errors so an update check never breaks startup.

## Relationships
- **Depends on:** `src/third_party/index.js` (`SerializedAXNode`, `Viewport`, `Target`, `semver`); `check-for-updates.ts` also imports `VERSION` from `src/version.js` and uses Node `child_process`/`fs`/`os`/`path`/`process`.
- **Used by:** `EmulationSettings` and friends are consumed broadly — `McpContext`, `McpPage`, `McpResponse`, `ToolHandler`, `TextSnapshot`, `SnapshotFormatter`, telemetry, and `bin/chrome-devtools-mcp-main.ts`. `PaginationOptions` is consumed by `McpResponse` and `tools/ToolDefinition.ts`. `checkForUpdates` is called only from the CLI entrypoints `src/bin/chrome-devtools-mcp-main.ts` and `src/bin/chrome-devtools.ts` (see the entrypoints layer for full coverage).

## Gotchas & non-obvious details
- **Two `types.ts` files.** `src/types.ts` (domain interfaces) and `src/utils/types.ts` (`PaginationOptions`) are distinct — `paginate()` imports the latter, but `McpResponse` re-exports/imports `PaginationOptions` from `./utils/types.js`. Don't conflate them.
- `EmulationSettings` is entirely optional fields — an empty object is valid and means "emulate nothing".
- `check-for-updates.ts` is intentionally fail-silent everywhere; absence of a warning does not mean you're up to date (network/cache could have failed). It uses a *cached* version, so the warning may lag by up to a day.
- The update check writes outside the repo (the user's home `~/.cache`) and spawns a detached subprocess — relevant when reasoning about side effects of merely starting the CLI.

## Update triggers
- A field is added/removed on any `src/types.ts` interface or on `PaginationOptions`.
- The update-check cache path, throttle window, env-var name, or subprocess path changes.
