# Tool Group: Console

> **Layer:** Tool system · **Sources:** `src/tools/console.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Tools to list console messages captured for the selected page and to fetch a single message by id. Category `DEBUGGING`, both `readOnlyHint: true`.

## Key files
- `src/tools/console.ts` — `list_console_messages`, `get_console_message`, and the `FILTERABLE_MESSAGE_TYPES` list.

## Key types & functions
Both are `definePageTool`, category `DEBUGGING`, `blockedByDialog: false`.

- `list_console_messages` (`definePageTool` factory) — schema `{pageSize?, pageIdx?, types?, includePreservedMessages?, serviceWorkerId?}`.
  - `pageSize` positive int, `pageIdx` 0-based int (omit → first page).
  - `types` is `zod.array(zod.enum(FILTERABLE_MESSAGE_TYPES))` — the standard `ConsoleMessageType` values (`log`, `debug`, `info`, `error`, `warn`, `dir`, `dirxml`, `table`, `trace`, `clear`, `startGroup`, `startGroupCollapsed`, `endGroup`, `assert`, `profile`, `profileEnd`, `count`, `timeEnd`, `verbose`) plus the synthetic `'issue'`.
  - `includePreservedMessages` (default false) → preserved messages over the last 3 navigations.
  - `serviceWorkerId` → filter to one service worker.
  - Description appends an extensions note when `categoryExtensions`.
- `get_console_message` — `{msgid}`; `response.attachConsoleMessage(msgid)`.

## How it works
- `list_console_messages` simply calls `response.setIncludeConsoleData(true, {pageSize, pageIdx, types, includePreservedMessages, serviceWorkerId})`. As with network, the actual message collection/formatting lives in the response layer ([McpResponse](../../03-context-state/mcp-response.md)) over the console [PageCollector](../../03-context-state/collectors.md).
- `get_console_message` calls `response.attachConsoleMessage(msgid)` to inline one message's full detail.
- `ConsoleResponseType = ConsoleMessageType | 'issue'` — `'issue'` covers DevTools "Issues" surfaced alongside console messages.

## Relationships
- **Depends on:** [McpResponse](../../03-context-state/mcp-response.md) (`setIncludeConsoleData`, `attachConsoleMessage`), console capture in [PageCollector](../../03-context-state/collectors.md), `ConsoleMessageType` (Puppeteer).
- **Used by:** agents debugging page errors; pairs with [script](script.md)/[snapshot](snapshot.md).

## Gotchas & non-obvious details
- Like network tools, these set flags on the `Response`; pagination and rendering happen in `response.handle`.
- `'issue'` is not a real Puppeteer console type — it's added to the filter union to expose DevTools issues.
- `serviceWorkerId` filtering ties into the extensions/service-worker collectors; meaningful mainly when extension/service-worker capture is active.

## Update triggers
- `FILTERABLE_MESSAGE_TYPES` changes, or a console tool is added.
- The `setIncludeConsoleData` options shape changes (see [tool-definition](../tool-definition.md) `Response`).
