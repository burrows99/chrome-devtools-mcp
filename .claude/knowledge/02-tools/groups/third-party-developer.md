# Third-party developer tools

> **Layer:** Tool system · **Sources:** `src/tools/thirdPartyDeveloper.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Bridges to "third-party developer tools" a page exposes for runtime introspection — tools registered on `window.__dtmcp.toolGroups`. One MCP tool lists them; another validates params and executes a named one. Experimental and off by default.

## Key files
- `src/tools/thirdPartyDeveloper.ts` — defines `list_3p_developer_tools` and `execute_3p_developer_tool`, plus the `ToolDefinition` / `ToolGroup` / `ToolGroups` types and the `Window.__dtmcp` global augmentation.

## Tools exposed
Both are page-scoped, `ToolCategory.THIRD_PARTY` (`'experimentalThirdParty'`, in `OFF_BY_DEFAULT_CATEGORIES`), `blockedByDialog: false`.
- `list_3p_developer_tools` — No params, `readOnlyHint: true`. Calls `response.setListThirdPartyDeveloperTools()`. The description tells callers they can run a listed tool via `execute_3p_developer_tool` OR via `evaluate_script` calling `window.__dtmcp.executeTool(toolName, params)` (useful for non-serializable returns or composition).
- `execute_3p_developer_tool` — `readOnlyHint: false`. Params: `toolName` (string), `params` (optional JSON-stringified string). Parses `params` (must be a JSON object), locates the tool across the page's tool groups, AJV-validates the params against the tool's `inputSchema`, then runs it.

## How it works
- `execute_3p_developer_tool`:
  1. `JSON.parse`s `params` if present; throws `Failed to parse params as JSON: ...` on bad JSON and requires the result be a non-null object.
  2. `request.page.getThirdPartyDeveloperTools()` returns `ToolGroups`; it searches each group's `tools` for a matching `name`, throwing `Tool ${toolName} not found` if absent.
  3. Compiles the tool's `inputSchema` (a `JSONSchema7`) with a fresh `ajv` instance and validates `params`; throws `Invalid parameters for tool ${toolName}: ...` on failure.
  4. Delegates to `request.page.executeThirdPartyDeveloperTool(toolName, params, response)` which runs the tool in the page and writes into the response.
- The page-side contract is the `window.__dtmcp` global: `toolGroups` (each tool has `name`/`description`/`inputSchema`/`execute`), `executeTool(name, args)`, and `stashedElements`.

## Relationships
- **Depends on:** `ContextPage.getThirdPartyDeveloperTools()` / `executeThirdPartyDeveloperTool()` (declared in `src/tools/ToolDefinition.ts`, which imports the `ToolGroups` type from this file); `ajv` and `JSONSchema7` (re-exported via `src/third_party/index.js`); `response.setListThirdPartyDeveloperTools()`.
- **Used by:** the MCP `ToolHandler`. Conceptually parallel to the [WebMCP tools](./webmcp.md) (list + execute page-exposed tools), but a different page-side protocol (`__dtmcp` vs. `page.webmcp`).

## Gotchas & non-obvious details
- Off by default — the `experimentalThirdParty` category is in `OFF_BY_DEFAULT_CATEGORIES`.
- Params are passed as a JSON string (`params`), not a structured object, and are validated client-side via AJV against the page-provided `inputSchema` before execution.
- A fresh `ajv` instance is created per call.
- There is a third execution path: `evaluate_script` + `window.__dtmcp.executeTool(...)`, recommended when results are non-serializable or need composing with other JS.
- This file also owns the `Window.__dtmcp` global type augmentation used elsewhere.

## Update triggers
- A tool is added/renamed or a param changes in `src/tools/thirdPartyDeveloper.ts`.
- The `window.__dtmcp` contract (`toolGroups` / `executeTool` / tool `inputSchema`) changes.
- The `experimentalThirdParty` category gating changes.
