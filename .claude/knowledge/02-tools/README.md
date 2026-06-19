# Tool System

> **Layer:** Tool system · **Sources:** `src/tools/ToolDefinition.ts`, `src/ToolHandler.ts`, `src/tools/categories.ts`, `src/tools/tools.ts`, `src/index.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
"Tools" are the discrete actions an MCP client (an AI agent) can invoke on this server. Each tool wraps a Chrome/Puppeteer/Lighthouse capability behind a typed schema, a category, and a handler. This subsystem defines what a tool *is* (`ToolDefinition`), how tools are grouped (`categories`), how they are collected (`createTools`), and how a single call is validated, dispatched, executed, and answered (`ToolHandler`).

## Key files
- `src/tools/ToolDefinition.ts` — `ToolDefinition` shape + `defineTool`/`definePageTool` factories; `Context`/`ContextPage`/`Response` interfaces tools may use.
- `src/ToolHandler.ts` — per-tool wrapper: enable/disable logic, arg validation, mutex, execution, telemetry.
- `src/tools/categories.ts` — the `ToolCategory` enum, labels, and which categories are off by default.
- `src/tools/tools.ts` — `createTools(args)` aggregates every group module into one sorted tool list.
- `src/tools/slim/tools.ts` — the reduced 3-tool "slim" surface.
- `src/index.ts` — `registerTool` loop that binds each `ToolHandler` to the MCP `server`.
- `src/tools/<group>.ts` — one module per tool group; each `export`s tool definitions.

## Key types & functions
- `ToolDefinition<Schema>` / `DefinedPageTool<Schema>` — declarative tool descriptors (`src/tools/ToolDefinition.ts`).
- `defineTool` / `definePageTool` — identity-style factories; `definePageTool` tags the tool `pageScoped: true`.
- `ToolHandler` — constructed per tool in `src/index.ts`; `.handle(params)` runs one call.
- `createTools(args)` — instantiates and alphabetically sorts all tools for the current CLI args.
- `ToolCategory` — `input`, `navigation`, `emulation`, `performance`, `network`, `debugging`, `extensions`, `experimentalThirdParty`, `memory`, `experimentalWebmcp`.

## How it works
1. `createTools(serverArgs)` picks the slim set or the full set, calls factory tools with `args`, and sorts by name.
2. For each tool, `src/index.ts:registerTool` builds a `ToolHandler`. If `shouldRegister` is false it is skipped (unless running via CLI). Otherwise it calls `server.registerTool(name, {description, inputSchema, annotations}, cb)` where `cb` delegates to `toolHandler.handle`.
3. On invocation, `ToolHandler.handle` rejects disabled tools and unknown args, acquires a global `Mutex`, resolves the `McpContext`, builds an `McpResponse` (or `SlimMcpResponse`), validates file paths, dispatches to the tool's `handler`, then serializes via `response.handle`.

```
client → server.registerTool cb → ToolHandler.handle → tool.handler(request, response, context) → response.handle() → CallToolResult
```

## Group index
Set A (documented here):
- [Input](groups/input.md) — click, fill, type, drag, key/dialog handling.
- [Pages & navigation](groups/pages-navigation.md) — list/select/new/close pages, navigate, resize, dialog, tab id.
- [Network](groups/network.md) — list/get network requests.
- [Console](groups/console.md) — list/get console messages.
- [Snapshot](groups/snapshot.md) — a11y text snapshot, wait_for text.
- [Screenshot](groups/screenshot.md) — take_screenshot.
- [Screencast](groups/screencast.md) — start/stop video recording (experimental).
- [Script](groups/script.md) — evaluate_script.

Set B (owned by the other agent — links may be stubs until written):
- [Performance](groups/performance.md) · [Memory](groups/memory.md) · [Emulation](groups/emulation.md) · [Extensions](groups/extensions.md) · [Lighthouse](groups/lighthouse.md) · [Third-party developer](groups/third-party-developer.md) · [WebMCP](groups/webmcp.md)

Core docs: [Tool definition](tool-definition.md) · [Tool handler](tool-handler.md) · [Categories & discovery](categories-and-discovery.md) · [Slim mode](slim-mode.md)

## Relationships
- **Depends on:** [McpContext](../03-context-state/mcp-context.md), [McpResponse](../03-context-state/mcp-response.md), Puppeteer/CDP via `src/third_party`.
- **Used by:** the MCP server entrypoint in `src/index.ts`.

## Gotchas & non-obvious details
- Public tool docs (`docs/tool-reference.md`, `docs/slim-tool-reference.md`) are auto-generated (`npm run gen`); do not hand-edit and cross-reference rather than duplicate.
- A tool's MCP `name` is the `name` field, not the `export` symbol (e.g. `export const screenshot` in `screenshot.ts` registers as `take_screenshot`).

## Update triggers
- A new `src/tools/<group>.ts` module is added (add to `tools.ts` and link a group file here).
- `ToolCategory`, `OFF_BY_DEFAULT_CATEGORIES`, or the registration flow in `src/index.ts` changes.
