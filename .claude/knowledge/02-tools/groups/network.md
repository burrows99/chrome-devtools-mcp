# Tool Group: Network

> **Layer:** Tool system · **Sources:** `src/tools/network.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Tools to list network requests captured for the selected page and to fetch a single request's details (headers/body), optionally saving bodies to disk. Category `NETWORK`.

## Key files
- `src/tools/network.ts` — `list_network_requests`, `get_network_request`, and the `FILTERABLE_RESOURCE_TYPES` enum list.

## Key types & functions
Both are `definePageTool`, category `NETWORK`.

- `list_network_requests` — `readOnlyHint: true`, `blockedByDialog: false`. Schema: `{pageSize?, pageIdx?, resourceTypes?, includePreservedRequests?}`.
  - `pageSize` positive int (omit → all), `pageIdx` 0-based int (omit → first page).
  - `resourceTypes` is `zod.array(zod.enum(FILTERABLE_RESOURCE_TYPES))` — `document`, `stylesheet`, `image`, `media`, `font`, `script`, `texttrack`, `xhr`, `fetch`, `prefetch`, `eventsource`, `websocket`, `manifest`, `signedexchange`, `ping`, `cspviolationreport`, `preflight`, `fedcm`, `other`.
  - `includePreservedRequests` (default false) → preserved requests over the last 3 navigations.
- `get_network_request` — `readOnlyHint: false`, `blockedByDialog: true`. Schema: `{reqid?, requestFilePath?, responseFilePath?}`; `verifyFilesSchema: ['requestFilePath','responseFilePath']`. If `reqid` omitted, returns whatever is selected in the DevTools Network panel.

## How it works
- `list_network_requests`: fetches `request.page.getDevToolsData()`, attaches it via `response.attachDevToolsData(data)`, and if `data.cdpRequestId` is present resolves it to a `reqid` via `context.resolveCdpRequestId(page, cdpRequestId)`. Then `response.setIncludeNetworkRequests(true, {pageSize, pageIdx, resourceTypes, includePreservedRequests, networkRequestIdInDevToolsUI: reqid})`. The actual request collection/formatting happens in the response layer ([McpResponse](../../03-context-state/mcp-response.md)) backed by the [PageCollector](../../03-context-state/collectors.md).
- `get_network_request`: if `reqid` provided → `response.attachNetworkRequest(reqid, {requestFilePath, responseFilePath})`. Otherwise resolves the currently-selected DevTools request (same `getDevToolsData` → `resolveCdpRequestId` path); if none, appends "Nothing is currently selected in the DevTools Network panel."
- File paths (`.network-request` / `.network-response`) let large bodies be written to disk instead of inlined; they are validated by `verifyFilesSchema` before the handler runs.

## Relationships
- **Depends on:** [ContextPage](../tool-definition.md) (`getDevToolsData`), [McpContext](../../03-context-state/mcp-context.md) (`resolveCdpRequestId`), [McpResponse](../../03-context-state/mcp-response.md) (`setIncludeNetworkRequests`, `attachNetworkRequest`, `attachDevToolsData`), network capture in [PageCollector](../../03-context-state/collectors.md).
- **Used by:** agents inspecting page traffic; complements [performance](performance.md) traces.

## Gotchas & non-obvious details
- These tools don't build the request list themselves — they set flags/ids on the `Response`; serialization and pagination occur in `response.handle`.
- `reqid` is an MCP-facing integer mapped from the CDP request id by `resolveCdpRequestId`; the selected-request path reuses DevTools' own selection.
- `get_network_request` is `blockedByDialog: true` (a fetch can be blocked by an open dialog) whereas `list_network_requests` is not.
- Redaction of headers is controlled server-wide (`setRedactNetworkHeaders` in `ToolHandler`), not per call.

## Update triggers
- `FILTERABLE_RESOURCE_TYPES` changes, or a network tool is added.
- The `setIncludeNetworkRequests` options shape changes (see [tool-definition](../tool-definition.md) `Response`).
