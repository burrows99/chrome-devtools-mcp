# Slim Mode

> **Layer:** Tool system · **Sources:** `src/tools/slim/tools.ts`, `src/tools/tools.ts`, `src/ToolHandler.ts`, `src/SlimMcpResponse.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Slim mode is a minimal tool surface — three tools instead of the full set — selected with the `--slim` CLI flag. It exists for clients/contexts that want a tiny, low-overhead automation API rather than the full DevTools toolbox.

## Key files
- `src/tools/slim/tools.ts` — the entire slim tool set (re-exported `screenshot`, `navigate`, `evaluate`).
- `src/tools/tools.ts` — `createTools` branches to `slimTools` when `args.slim`.
- `src/SlimMcpResponse.ts` — the trimmed `Response` impl used under slim.

## Key types & functions
- `screenshot` (MCP name `screenshot`) — DEBUGGING; no params; PNG to a temp file, returns the path.
- `navigate` (MCP name `navigate`) — NAVIGATION; `{url}`; goto with a fixed 30s timeout, auto-accepts `beforeunload`.
- `evaluate` (MCP name `evaluate`) — DEBUGGING; `{script}`; `pptrPage.evaluate(script)` and JSON-stringifies the result.

## How it works
- `createTools(args)` uses `Object.values(slimTools)` exclusively when `args.slim` is set — none of the full group modules are loaded.
- The slim tools are deliberately simpler than their full counterparts:
  - slim `navigate` takes only a `url` string and hardcodes `timeout: 30_000`; the full `navigate_page` adds `type` (url/back/forward/reload), `ignoreCache`, `handleBeforeUnload`, `initScript`, allowlist, and timeout.
  - slim `screenshot` is param-less and always PNG-to-temp-file; the full `take_screenshot` supports format/quality/uid/fullPage/filePath and inline image attachment.
  - slim `evaluate` takes a raw script string; the full `evaluate_script` takes a function declaration + element-uid args + service-worker routing + file output.
- In slim mode `ToolHandler` never adds `pageIdSchema` (page-id routing is disabled regardless of the flag) and builds a `SlimMcpResponse` instead of `McpResponse`.

## Relationships
- **Depends on:** [ToolDefinition](tool-definition.md) (`definePageTool`), [categories](categories-and-discovery.md), [SlimMcpResponse](../03-context-state/mcp-response.md).
- **Used by:** `createTools` selection in [categories & discovery](categories-and-discovery.md); contrasts with full [script](groups/script.md), [pages](groups/pages-navigation.md), [screenshot](groups/screenshot.md) tools.

## Gotchas & non-obvious details
- Slim tool MCP names collide conceptually with full ones but are distinct: slim exposes `navigate`/`screenshot`/`evaluate`, while the full set uses `navigate_page`/`take_screenshot`/`evaluate_script`. They are never both registered (the set is chosen once at startup).
- All three slim tools are `definePageTool` (page-scoped) but page-id routing stays off, so they always act on the selected page.
- The generated `docs/slim-tool-reference.md` documents exactly these three tools.

## Update triggers
- A tool is added to or removed from `src/tools/slim/tools.ts`.
- The slim branch in `createTools` or the slim response wiring in `ToolHandler` changes.
