# McpResponse & SlimMcpResponse

> **Layer:** Context & state · **Sources:** `src/McpResponse.ts`, `src/SlimMcpResponse.ts`, `src/tools/ToolDefinition.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`McpResponse` is the output builder for a single tool call. Tools call declarative `set*`/`attach*`/`include*` methods during the handler; later `handle()` gathers the requested data from `McpContext`, runs it through formatters, and produces the MCP result: `{content: (TextContent | ImageContent)[], structuredContent: object}`. `SlimMcpResponse` is a minimal variant for `--slim` mode.

## Key files
- `src/McpResponse.ts` — `McpResponse implements Response` (`:200`) plus `getToolGroups`, `replaceHtmlElementsWithUids`, page-title helpers.
- `src/SlimMcpResponse.ts` — `SlimMcpResponse extends McpResponse` (`:15`).
- `src/tools/ToolDefinition.ts` — the `Response` interface (`:102`).

## Key types & functions
- `Response` — the interface tools see (`appendResponseLine`, `includeSnapshot`, `setIncludeNetworkRequests`, `setIncludeConsoleData`, `attach*`, `setHeapSnapshot*`, …) (`src/tools/ToolDefinition.ts:102`).
- `McpResponse.handle(toolName, context, useToon?)` — collects data + calls `format()` (`:524`).
- `McpResponse.format(toolName, context, data, useToon)` — assembles `response[]` text lines and `structuredContent` (`:777`).
- `#dataWithPagination(data, pagination?)` — wraps `paginate()` and emits "Showing X-Y of Z" lines (`:1304`).

## How it works
A response is a deferred spec, not eager output. The handler sets flags/IDs:
- `includeSnapshot(params?)` → snapshot; `setIncludePages(bool)` → page list (and extension pages/SWs if the extensions category is enabled).
- `setIncludeNetworkRequests(bool, opts)` / `setIncludeConsoleData(bool, opts)` → list data with optional pagination, type/resource filtering, preserved-data and service-worker-id options.
- `attachNetworkRequest(id, opts?)` / `attachConsoleMessage(id)` → one detailed item.
- `setHeapSnapshot*`, `attachTraceSummary/Insight`, `attachLighthouseResult`, `setListExtensions`, `setListThirdPartyDeveloperTools`, `setListWebMcpTools`, `attachImage`, `attachWaitForResult`, `setError`.

Then `handle()` (run by `ToolHandler` after the handler):
1. Re-snapshots pages / extension SWs if `setIncludePages` requested it.
2. If a snapshot was requested, rebuilds `page.textSnapshot` via `TextSnapshot.create` and wraps it in a `SnapshotFormatter` (or saves to file).
3. Fetches detailed network request / console message / lists from the relevant collector and builds the matching formatter (`NetworkFormatter`, `ConsoleFormatter`, `IssueFormatter`).
4. Calls `format()`, which appends text lines to `response[]` and mirrors each into `structuredContent`, returning `{content, structuredContent}`.

## Output shape: text vs structured
Every section is emitted **twice**: human-readable lines pushed onto `response[]` (joined with `\n` into one `TextContent`) and a structured field on `structuredContent` (e.g. `structuredContent.networkRequests`, `.snapshot`, `.consoleMessages`, `.pagination`, `.dialog`, `.viewport`). The `format()` method also auto-emits emulation/dialog state read off `#page` (network conditions, geolocation, viewport, user agent, CPU throttling, color scheme, open dialog) — tools don't have to request these. `structuredContent` is only returned to the client when `experimentalStructuredContent` is set (`ToolHandler`).

## TOON & pagination
- When `useToon` is true (`experimentalToonFormat`), list-like sections (snapshot, heap data, network requests, console messages) emit a `toonEncode(...)` block instead of the formatters' `.toString()` — more compact for LLMs.
- Pagination is applied per list section via `#dataWithPagination` → `paginate()` ([Utilities](../08-utilities/README.md)), producing "Showing 1-N of M (Page p of t)" plus next/previous hints and a shared `structuredContent.pagination`.

## SlimMcpResponse
`SlimMcpResponse.handle` ignores all the rich collection logic and returns only the joined `responseLines` as text (and the same text object as `structuredContent`). Used in `--slim` mode (also disables `pageId` routing). It still inherits all the `set*`/`attach*` setters (tools call them harmlessly).

## Relationships
- **Depends on:** [Formatters](../05-formatters/README.md) (`SnapshotFormatter`, `NetworkFormatter`, `ConsoleFormatter`, `IssueFormatter`, `HeapSnapshotFormatter`), [McpContext](mcp-context.md) (data + `saveFile`), [McpPage](mcp-page.md) (emulation/dialog/snapshot), [Performance & memory](../06-performance-memory/README.md) (`getTraceSummary`, `getInsightOutput`), [Utilities](../08-utilities/README.md) (`paginate`), `toonEncode` from third_party.
- **Used by:** [Entrypoints](../01-entrypoints/README.md) (`ToolHandler` constructs it and calls `handle`), [Tool system](../02-tools/README.md) (handlers call its setters).

## Gotchas & non-obvious details
- Snapshot rebuild happens inside `handle()`, not when `includeSnapshot()` is called — the snapshot reflects post-action DOM.
- `replaceHtmlElementsWithUids` rewrites third-party tool input schemas so `HTMLElement`-typed params become `{uid: string}` (LLMs pass UIDs, not DOM nodes).
- `getToolGroups` dispatches a `devtoolstooldiscovery` event into the page and collects responding tool groups (sync or microtask), installing `window.__dtmcp.executeTool`.
- `attachWaitForResult` only adds a `navigatedToUrl` line/field if a navigation occurred.

## Update triggers
- New response section or `Response` method, change to text/`structuredContent` shape, TOON encoding, pagination format, or the slim-mode contract.
