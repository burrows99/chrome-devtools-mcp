# Tool Handler

> **Layer:** Tool system · **Sources:** `src/ToolHandler.ts`, `src/index.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`ToolHandler` is the per-tool runtime wrapper around a `ToolDefinition`. It decides whether the tool should be registered, computes its final input schema, and on each invocation validates args, serializes execution behind a mutex, resolves the context/response, dispatches to the tool handler, and emits telemetry.

## Key files
- `src/ToolHandler.ts` — the `ToolHandler` class plus gating helpers (`buildFlag`, `getToolStatusInfo`, etc.).
- `src/index.ts` — `registerTool` constructs one `ToolHandler` per tool and wires `.handle` as the MCP callback.

## Key types & functions
- `class ToolHandler` — fields `inputSchema`, `registeredInputSchema`, `shouldRegister`, private `disabledReason`; method `handle(params)`.
- `buildFlag(category)` — maps a category to its CLI flag name, e.g. `network` → `categoryNetwork`.
- `getToolStatusInfo(tool, serverArgs)` — returns `{disabled, reason?}` from category + condition checks.
- `isPageScopedTool(tool)` — type guard on `pageScoped === true`.
- `buildUnknownArgumentsMessage(...)` — friendly error listing unexpected vs. expected args.

## How it works
**Construction** (in `src/index.ts:registerTool`):
- Resolves `{disabled, reason}` via `getToolStatusInfo`. A category is disabled if it's in `OFF_BY_DEFAULT_CATEGORIES` and its flag is unset, or if its flag is explicitly `false`. Each `annotations.conditions[]` flag must also be truthy.
- `shouldRegister = !(disabled && !serverArgs.viaCli)` — disabled tools are still registered when running via CLI (so the CLI can report them), but hidden from a normal MCP server.
- `inputSchema` = the tool schema, prefixed with `pageIdSchema` only when the tool is page-scoped **and** `experimentalPageIdRouting` **and** not `slim`. `registeredInputSchema = zod.object(inputSchema).passthrough()`.

**Invocation** (`handle(params)`):
1. If `disabledReason` set → return an `isError` text result.
2. `unknownArgumentNames` → any param not in `inputSchema` → return an `isError` message (`.passthrough()` lets them through Zod, so this check is what rejects them).
3. Acquire the shared `toolMutex` (serializes all tool calls server-wide).
4. `getContext()` → `McpContext`; `await context.detectOpenDevToolsWindows()`.
5. Build `SlimMcpResponse` (if `slim`) else `McpResponse`; set `redactNetworkHeaders`.
6. Run `verifyFilesSchema` paths through `context.validatePath`.
7. **Page-scoped:** resolve the page (`getPageById` under page-id routing, else `getSelectedMcpPage`), `response.setPage(page)`, and if `blockedByDialog` call `page.throwIfDialogOpen()`; then call `tool.handler({params, page}, response, context)`. **Non-page:** call `tool.handler({params}, response, context)`.
8. Handler errors are caught into `response.setError(err)` (not thrown) so partial output still serializes.
9. `response.handle(toolName, context, experimentalToonFormat)` → `{content, structuredContent}`. Result gets `isError` if `response.error`; `structuredContent` attached only under `experimentalStructuredContent`.
10. `finally`: log `ClearcutLogger.logToolInvocation(...)` with bucketized latency, then release the mutex guard.

A second `try/catch` around the whole body converts unexpected throws (e.g. from `getContext`) into an `isError` text result, appending `Cause:` when present.

## Relationships
- **Depends on:** [ToolDefinition](tool-definition.md), [categories](categories-and-discovery.md), [McpContext](../03-context-state/mcp-context.md), [McpResponse](../03-context-state/mcp-response.md), `Mutex`, `ClearcutLogger`.
- **Used by:** `src/index.ts` registration loop.

## Gotchas & non-obvious details
- The global `toolMutex` means tool calls never run concurrently — one in-flight call blocks the next.
- Unknown-arg rejection is manual, not Zod: `registeredInputSchema` uses `.passthrough()`, so extra keys pass schema validation and are caught by `unknownArgumentNames`.
- Page-id routing is silently ignored in slim mode even if the flag is set.
- Handler exceptions become response errors via `setError`; only context-setup failures hit the outer catch.

## Update triggers
- Gating logic (category/condition/slim/CLI) changes.
- `handle`'s execution order or telemetry changes.
- A new response mode beyond `McpResponse`/`SlimMcpResponse` is added.
