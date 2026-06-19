# Files & Pagination Utilities

> **Layer:** Utilities & shared types · **Sources:** `src/utils/files.ts`, `src/utils/pagination.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Filesystem path helpers (temp files, extension normalization, symlink-resolved canonical paths) and a generic array pagination function with navigation metadata.

## Key files
- `src/utils/files.ts` — path/filesystem helpers built on `node:fs/promises`, `node:os`, `node:path`.
- `src/utils/pagination.ts` — generic `paginate()` plus the `PaginationResult` shape.

## Key functions / types
- `getTempFilePath(filename: string): Promise<string>` (`files.ts`) — creates a fresh unique temp dir via `fs.mkdtemp(os.tmpdir()/'chrome-devtools-mcp-')` and returns `dir/filename`. A new directory per call (no collisions).
- `ensureExtension(filepath, extension: \`.${string}\`): string` (`files.ts`) — strips the existing extension and appends `extension`. Template-literal type forces the caller to pass a leading dot (e.g. `'.png'`). Note: if the path has no extension it still appends.
- `resolveCanonicalPath(filePath: string): Promise<string>` (`files.ts`) — resolves to absolute, then `fs.realpath` to follow all symlinks. If the target does not exist (`ENOENT`), it walks up to the nearest existing ancestor, canonicalizes that, and re-joins the missing trailing segments — so it returns a canonical path even for a not-yet-created file. Re-throws non-`ENOENT` errors and re-throws if it walks all the way to the filesystem root without resolving.
- `paginate<Item>(items, options?): PaginationResult<Item>` (`pagination.ts`) — slices `items` to a page and returns navigation metadata.
- `PaginationResult<Item>` (`pagination.ts`) — `{ items, currentPage, totalPages, hasNextPage, hasPreviousPage, startIndex, endIndex, invalidPage }`.

## How it works
`paginate()` short-circuits when `options` is absent or both `pageSize` and `pageIdx` are undefined (`noPaginationOptions`): it returns all items as `currentPage: 0`, `totalPages: 1`, `endIndex: total`. Otherwise `pageSize` defaults to `DEFAULT_PAGE_SIZE = 20`; `totalPages = max(1, ceil(total / pageSize))`. `resolvePageIndex()` validates `pageIdx`: undefined → page 0; out of range (`< 0` or `>= totalPages`) → **falls back to page 0 and sets `invalidPage: true`** (it does not throw — callers inspect the flag). `startIndex = currentPage * pageSize`; `endIndex = startIndex + pageItems.length` (so `endIndex` reflects the actual slice, not the requested window).

## Relationships
- **Depends on:** `files.ts` → Node `fs`/`os`/`path`. `pagination.ts` → `PaginationOptions` from `src/utils/types.ts` (see [shared-types.md](shared-types.md)).
- **Used by:**
  - `files.ts` → `src/McpContext.ts` (all three), `src/tools/memory.ts` + `src/tools/screencast.ts` (`ensureExtension`), `src/daemon/client.ts` (`getTempFilePath`).
  - `pagination.ts` → `src/McpResponse.ts` (`#dataWithPagination` wraps `paginate()` for list/record responses, including the heap-snapshot record path).

## Gotchas & non-obvious details
- `paginate` never throws on a bad page index — it silently returns page 0 with `invalidPage: true`. Forgetting to surface that flag hides bad input from the user.
- `endIndex` is the end of the *returned* slice; on the last (partial) page `endIndex - startIndex < pageSize`.
- `resolveCanonicalPath` is the security-relevant one: it normalizes symlinks so path-confinement checks in `McpContext` operate on the real location, and it works even when the file doesn't exist yet (e.g. an output path about to be written).
- `getTempFilePath` leaks temp directories — it creates a new dir every call and never cleans up; cleanup (if any) is the caller's responsibility.

## Update triggers
- `paginate` default page size or invalid-page handling changes.
- `resolveCanonicalPath`'s ENOENT-walk behavior changes.
- A new path helper is added to `files.ts`.
