# Context & state

> **Layer:** Context & state · **Sources:** `src/McpContext.ts`, `src/McpPage.ts`, `src/McpResponse.ts`, `src/SlimMcpResponse.ts`, `src/PageCollector.ts`, `src/ServiceWorkerCollector.ts`, `src/TextSnapshot.ts`, `src/Mutex.ts`, `src/WaitForHelper.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The runtime state every tool operates on. `McpContext` is the single hub that owns the browser, all pages, the data collectors, and file/heap-snapshot/emulation state; `McpPage` wraps each Puppeteer page; `McpResponse` accumulates and serializes tool output. This is the most-connected layer in the codebase — `McpContext` bridges roughly 16 subsystems.

## Key files
- `src/McpContext.ts` — central state hub; implements the `Context` interface tools see. See [mcp-context.md](mcp-context.md).
- `src/McpPage.ts` — per-page wrapper (`ContextPage`) around a Puppeteer `Page`. See [mcp-page.md](mcp-page.md).
- `src/McpResponse.ts` / `src/SlimMcpResponse.ts` — output builder/serializer. See [mcp-response.md](mcp-response.md).
- `src/PageCollector.ts` — `PageCollector`, `NetworkCollector`, `ConsoleCollector`. See [collectors.md](collectors.md).
- `src/ServiceWorkerCollector.ts` — `ServiceWorkerConsoleCollector` for extension SWs. See [collectors.md](collectors.md).
- `src/TextSnapshot.ts` — accessibility-tree text snapshot + UID system. See [text-snapshot.md](text-snapshot.md).
- `src/Mutex.ts` / `src/WaitForHelper.ts` — serialization & navigation/DOM waiting. See [concurrency.md](concurrency.md).

## Key types & functions
- `McpContext implements Context` — owns browser + all subsystems (`src/McpContext.ts:74`).
- `McpPage implements ContextPage` — per-page state (`src/McpPage.ts:42`).
- `McpResponse implements Response` — declarative output spec → `{content, structuredContent}` (`src/McpResponse.ts:200`).
- `Context` / `ContextPage` / `Response` — the read-only interfaces tools actually depend on (`src/tools/ToolDefinition.ts:180`, `:279`, `:102`).

## How it works
The lifecycle of one tool call (driven by `ToolHandler.handle`, `src/ToolHandler.ts:177`):

1. **Serialize.** A single shared `Mutex` (`src/index.ts:157`) is acquired so only one tool runs at a time. `await this.toolMutex.acquire()`.
2. **Resolve context.** `getContext()` returns the singleton `McpContext`; then `context.detectOpenDevToolsWindows()` refreshes DevTools page handles.
3. **Build a response.** A fresh `McpResponse` (or `SlimMcpResponse` in `--slim` mode) is created per call.
4. **Resolve the page.** For page-scoped tools the handler gets either `getPageById(pageId)` (when `experimentalPageIdRouting`) or `getSelectedMcpPage()`; the page is set on the response and `throwIfDialogOpen()` runs if `blockedByDialog`.
5. **Run the handler.** The tool mutates state on `context` and declares output on `response` (e.g. `response.includeSnapshot()`, `response.setIncludeNetworkRequests(true)`). Thrown errors are caught into `response.setError`.
6. **Serialize output.** `response.handle(toolName, context, useToon)` re-snapshots pages if requested, pulls data from the collectors, runs formatters, and returns MCP `content` (text + images) plus `structuredContent`.
7. **Release.** The mutex guard is disposed in `finally`.

Two ID systems run through this layer:
- **Page IDs** (`McpPage.id`, monotonically `#nextPageId`) identify pages to the LLM.
- **UIDs** (`snapshotId_idx`) identify accessibility nodes within a `TextSnapshot`; **stable resource IDs** (`stableIdSymbol`) identify network requests and console messages within a collector.

## Relationships
- **Depends on:** [Browser & CDP](../04-browser-cdp/README.md) (Puppeteer `Browser`/`Page`, CDP sessions, `UniverseManager`), [Formatters](../05-formatters/README.md) (Snapshot/Network/Console/Issue/HeapSnapshot formatters), [Performance & memory](../06-performance-memory/README.md) (`HeapSnapshotManager`, trace parsing), [Utilities](../08-utilities/README.md) (`paginate`, `createIdGenerator`, file helpers).
- **Used by:** [Tool system](../02-tools/README.md) — every tool handler receives `(request, response, context)`; [Entrypoints](../01-entrypoints/README.md) (`ToolHandler`, `src/index.ts` wires the singleton context + mutex).

## Gotchas & non-obvious details
- `McpContext` is constructed privately; always create it via `McpContext.from(...)` so `#init()` (collector + universe-manager init) runs.
- The whole context is a single in-memory singleton per server connection — there is no per-page isolation of collectors; data is keyed by `Page` inside each collector.
- `dispose()` intentionally does **not** close isolated browser contexts (the browser may outlive the connection).

## Update triggers
- A new data collector or owned subsystem is added to `McpContext`.
- The tool-call lifecycle in `ToolHandler` changes (mutex, page resolution, response handling).
- Response serialization (text vs `structuredContent`, TOON encoding) changes.
- The UID / stable-ID scheme changes.
