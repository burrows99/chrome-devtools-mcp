# WebMCP tools

> **Layer:** Tool system · **Sources:** `src/tools/webmcp.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Bridges to WebMCP tools that a page exposes natively through Puppeteer's `page.webmcp` API. One MCP tool lists them; another executes a named one. Experimental and off by default.

## Key files
- `src/tools/webmcp.ts` — defines `list_webmcp_tools` and `execute_webmcp_tool` (both page-scoped).

## Tools exposed
Both use `ToolCategory.WEBMCP` (`'experimentalWebmcp'`, in `OFF_BY_DEFAULT_CATEGORIES`), `blockedByDialog: false`.
- `list_webmcp_tools` — No params, `readOnlyHint: true`. Calls `response.setListWebMcpTools()` (rendering reads the page's WebMCP tools).
- `execute_webmcp_tool` — `readOnlyHint: false`. Params: `toolName` (string), `input` (optional JSON-stringified parameters). Parses `input`, finds the tool in `page.pptrPage.webmcp.tools()`, runs `tool.execute(input)`, and appends the `{status, output, errorText}` result as pretty JSON.

## How it works
- `execute_webmcp_tool`:
  1. If `input` is present, `JSON.parse`s it; throws `Failed to parse input as JSON: ...` on bad JSON and requires a non-null object (else `Parsed input is not an object`).
  2. `request.page.pptrPage.webmcp.tools()` returns the page's WebMCP tools; `.find(t => t.name === toolName)`; throws `Tool ${toolName} not found` if absent.
  3. `const {status, output, errorText} = await tool.execute(input);` then `response.appendResponseLine(JSON.stringify({status, output, errorText}, null, 2))`.
- Unlike third-party developer tools, there is no AJV pre-validation here — params are passed straight to `tool.execute`, and the page-side WebMCP runtime owns validation/errors (surfaced via `status`/`errorText`).

## Relationships
- **Depends on:** Puppeteer's `page.webmcp` API (`.tools()`, each tool's `.name` / `.execute()`); `response.setListWebMcpTools()` declared in `src/tools/ToolDefinition.ts`.
- **Used by:** the MCP `ToolHandler`. Conceptually parallel to the [third-party developer tools](./third-party-developer.md) (list + execute page-exposed tools) but uses the native `page.webmcp` protocol instead of `window.__dtmcp`.

## Gotchas & non-obvious details
- Off by default — `experimentalWebmcp` is in `OFF_BY_DEFAULT_CATEGORIES`.
- Params arrive as a JSON string (`input`), not a structured object; no client-side schema validation is performed (contrast with `execute_3p_developer_tool`).
- The full `{status, output, errorText}` triple from the page tool is echoed verbatim as JSON — callers should inspect `status`/`errorText`, not assume success.
- Relies on the browser/Puppeteer build exposing `page.webmcp`; without it these tools cannot resolve any tools.

## Update triggers
- A tool is added/renamed or a param changes in `src/tools/webmcp.ts`.
- The Puppeteer `page.webmcp` API (`tools()` / `execute()` / result shape) changes.
- The `experimentalWebmcp` category gating changes.
