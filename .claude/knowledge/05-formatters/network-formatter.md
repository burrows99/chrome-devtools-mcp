# Network Formatter

> **Layer:** Formatters · **Sources:** `src/formatters/NetworkFormatter.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Formats Puppeteer `HTTPRequest`/`HTTPResponse` objects into concise list rows (method/url/status) or detailed single-request output (headers, bodies, failure, redirect chain). Handles body truncation, optional header redaction, optional saving of large bodies to disk, and consistent redirect-chain ordering.

## Key files
- `NetworkFormatter.ts` — the formatter plus text-conversion free functions and `getSizeLimitedString`.

## Key types & functions
- `NetworkFormatter` — main class; `private` constructor, built via the async factory.
- `NetworkFormatter.from(request, options)` — static async factory; calls `#loadDetailedData()` only when `options.fetchData` is set.
- `NetworkFormatterOptions` — `{requestId?, selectedInDevToolsUI?, requestIdResolver?, fetchData?, requestFilePath?, responseFilePath?, saveFile?, redactNetworkHeaders}`.
- `NetworkRequestConcise` / `NetworkRequestDetailed` — internal JSON shapes (detailed extends concise, adding headers, bodies, file paths, `failure?`, and a `redirectChain?: NetworkRequestConcise[]`).
- `#getStatusFromRequest` — derives `status`: HTTP status code, else `failure.errorText`, else `'pending'`.
- `#getFormattedResponseBody` — UTF-8 check via `node:buffer` `isUtf8`; returns `<empty response>`, `<binary data>`, or `<not available anymore>` as appropriate.

## How it works
**Concise** (`toString()` → `convertNetworkRequestConciseToString`):
```
reqid=<requestId> <method> <url> [<status>][ [selected in the DevTools Network panel]]
```
**Detailed** (`toStringDetailed()` → `converNetworkRequestDetailedToStringDetailed`): markdown sections `## Request <url>`, `Status:`, `### Request Headers` (`- name:value`), `### Request Body`, `### Response Headers`, `### Response Body`, `### Request failed with`, `### Redirect chain` (indented concise rows).

**Detail fetching** (`#loadDetailedData`, gated by `fetchData`):
- **Request body** — `postData()` then `fetchPostData()`. If `requestFilePath` set, the body is saved via `options.saveFile(...)` (`.network-request` extension) and only the path is kept; otherwise size-limited inline. Missing data → `<Request body not available anymore>`.
- **Response body** — same two-track logic (`.network-response`; `<Response body not available anymore>`). Inline path runs through `#getFormattedResponseBody`.
- Size cap: `BODY_CONTEXT_SIZE_LIMIT = 10000`; over-limit strings get `... <truncated>` appended (`getSizeLimitedString`).

**Header redaction** — when `redactNetworkHeaders` is true, request and response headers pass through DevTools `NetworkRequestFormatter.sanitizeHeaders` (`#redactNetworkHeaders`, applied in `toJSONDetailed`).

**Redirect chain** — `toJSONDetailed()` reads `request.redirectChain()`, **reverses it once**, and formats each hop with a child `NetworkFormatter` (resolving its id via `requestIdResolver`). The text converter must **not** reverse again — there is an explicit comment that doing so would make the text contradict `structuredContent`.

## Relationships
- **Depends on:** Puppeteer `HTTPRequest`/`HTTPResponse` and DevTools `NetworkRequestFormatter.sanitizeHeaders` via `src/third_party/index.js`; `node:buffer` `isUtf8`; the caller-supplied `saveFile` (from `McpContext`) and `requestIdResolver` (stable-id lookup).
- **Used by:** [`McpResponse`](../03-context-state/) — list path uses `from({fetchData:false})` + `toString()`/`toJSON()` (paginated, optionally TOON-encoded); drill-down uses `from({fetchData:true, requestIdResolver, requestFilePath, responseFilePath})` + `toStringDetailed()`/`toJSONDetailed()`.

## Gotchas & non-obvious details
- Redirect-chain ordering is the canonical "text must match JSON" example — reverse exactly once, in JSON.
- File-save vs inline is chosen **per body** by whether `requestFilePath`/`responseFilePath` is provided; if a path is set but `saveFile` is missing, it throws.
- Response-buffer and save failures are swallowed; a failed save leaves `responseBody = <Response body not available anymore>` rather than a path.
- `// TODO truncate the URL` — concise URLs are currently emitted un-truncated.
- Note the misspelled symbol names kept as-is: `formatHeadlers`, `converNetworkRequestDetailedToStringDetailed`.

## Update triggers
- Concise/detailed text format or section headers change.
- `BODY_CONTEXT_SIZE_LIMIT`, truncation marker, or binary/empty sentinels change.
- Redirect-chain ordering logic changes (keep text and JSON in sync).
- Header-redaction or file-save behavior changes.
