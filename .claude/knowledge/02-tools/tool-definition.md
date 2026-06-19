# Tool Definition

> **Layer:** Tool system · **Sources:** `src/tools/ToolDefinition.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Defines the declarative shape of every tool, the helper factories used to author tools, and the narrow `Context`/`ContextPage`/`Response` interfaces a tool handler is allowed to touch. It is the contract between a tool module and the rest of the server.

## Key files
- `src/tools/ToolDefinition.ts` — all the types/factories below; no runtime logic beyond the factories and a few shared schema/transform helpers.

## Key types & functions
- `BaseToolDefinition<Schema>` — `name`, `description`, `annotations`, `schema` (a Zod raw shape), `blockedByDialog`, `verifyFilesSchema`.
- `ToolDefinition<Schema>` — extends base, adds `handler(request, response, context)`.
- `DefinedPageTool<Schema>` — a page-scoped tool; `handler`'s `request` also carries a resolved `page: ContextPage`, and `pageScoped: true` is set.
- `defineTool` / `definePageTool` — author-facing factories. Each accepts either a plain definition object or a factory `(args?: ParsedArguments) => definition`, enabling tools whose schema/description depend on CLI flags.
- `Context` / `ContextPage` / `Response` — the only surface tools see of `McpContext` / `McpPage` / `McpResponse` (deliberately narrowed: "Only add methods used by tools/*").
- Shared schema helpers: `pageIdSchema`, `timeoutSchema` (coerces `<= 0` to `undefined`), `viewportTransform`, `geolocationTransform`, and the `CLOSE_PAGE_ERROR` constant.

## How it works
A tool is just an object. Annotations describe metadata the MCP client sees and the server uses for gating:

```ts
annotations: {
  title?: string;
  category: ToolCategory;   // drives enable/disable + docs grouping
  readOnlyHint: boolean;    // true = does not modify environment
  conditions?: string[];    // experimental CLI flags that must be on
}
```

- `schema` is a `zod.ZodRawShape` (a map of param name → Zod type), not a `zod.object`. `ToolHandler` wraps it in `zod.object(schema).passthrough()` for registration.
- `blockedByDialog: true` makes the handler refuse to run while a JS dialog is open on the page (`ToolHandler` calls `page.throwIfDialogOpen()`).
- `verifyFilesSchema: Array<keyof Schema>` lists param names holding file paths; `ToolHandler` runs `context.validatePath` on each before the handler executes (e.g. `['filePath']`, `['requestFilePath','responseFilePath']`).
- The factory form lets a tool conditionally add params, e.g. `evaluate_script` adds `pageId` only when `experimentalPageIdRouting` is set.

`Response` is a builder: handlers call methods like `appendResponseLine`, `includeSnapshot`, `attachImage`, `attachNetworkRequest`, `setIncludePages`, `attachWaitForResult` rather than returning a value. The handler returns `Promise<void>`.

## Relationships
- **Depends on:** Zod (`src/third_party`), `ToolCategory` (`categories.ts`), Puppeteer types.
- **Used by:** every `src/tools/<group>.ts` module; [ToolHandler](tool-handler.md) consumes the shape; [McpResponse](../03-context-state/mcp-response.md) implements `Response`.

## Gotchas & non-obvious details
- `readOnlyHint` is `false` on several "read-ish" tools (`take_snapshot`, `take_screenshot`) *because they accept a `filePath`* that writes to disk — the comment in source says so explicitly.
- `schema: {}` is a valid empty shape (e.g. `screenshot` slim tool, `screencast_stop`).
- `definePageTool` does not by itself add `pageId` to the schema — that is `ToolHandler`'s job under `experimentalPageIdRouting` (and never in slim mode).
- `Context`/`ContextPage` are `Readonly<{...}>` interface mirrors; the real impls (`McpContext`, `McpPage`) have many more methods not exposed here.

## Update triggers
- A field is added to `BaseToolDefinition` or `annotations`.
- A method is added to the `Context`, `ContextPage`, or `Response` interfaces.
- A new shared schema/transform helper is exported here.
