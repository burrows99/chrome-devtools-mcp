# Tool Group: Pages & Navigation

> **Layer:** Tool system · **Sources:** `src/tools/pages.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Tools to enumerate and switch between open pages (tabs), open/close pages, navigate (url/back/forward/reload), resize the window, and fetch a CDP tab id. Most are NAVIGATION category; `resize_page` is EMULATION and `handle_dialog` (documented in [input](input.md)) is INPUT.

## Key files
- `src/tools/pages.ts` — all tools below plus `handle_dialog` and the shared `navigateWithInterception` helper.

## Key types & functions
- `list_pages` (`defineTool` factory) — NAVIGATION, `readOnlyHint: true`, no params. Calls `response.setIncludePages(true)` + `setListThirdPartyDeveloperTools()` + `setListWebMcpTools()`. Description mentions extension service workers when `categoryExtensions`.
- `select_page` (`defineTool`) — NAVIGATION, read-only. `{pageId, bringToFront?}`; `context.getPageById` → `context.selectPage`; optional `pptrPage.bringToFront()`.
- `close_page` (`defineTool`) — NAVIGATION. `{pageId}`; `context.closePage`. Catches `CLOSE_PAGE_ERROR` (last page) and reports it instead of throwing.
- `new_page` (`defineTool` factory) — NAVIGATION. `{url, background?, isolatedContext?, allowList?(exp), timeout?}`; `context.newPage(background, isolatedContext)` then `navigateWithInterception`.
- `navigate_page` (`definePageTool` factory) — NAVIGATION. `{type?('url'|'back'|'forward'|'reload'), url?, ignoreCache?, handleBeforeUnload?('accept'|'decline'), initScript?, allowList?(exp), timeout?}`.
- `resize_page` (`definePageTool`) — **EMULATION**. `{width, height}`; normalizes window state then `pptrPage.resize({contentWidth, contentHeight})`.
- `get_tab_id` (`definePageTool`) — NAVIGATION, read-only, condition `experimentalInteropTools`. `{pageId}`; reads `(pptrPage as CdpPage)._tabId` → `response.setTabId`.
- `handle_dialog` — INPUT category; see [input.md](input.md).

## How it works
- **Page list freshness:** navigation tools end with `response.setIncludePages(true)` (and the third-party/webmcp list setters) so every result re-emits the current page list and available delegated tools.
- **`navigate_page`:** requires a `type` or `url` (defaults `type='url'`). Registers a `beforeunload` dialog handler honoring `handleBeforeUnload` (default accept). If `initScript` given, `evaluateOnNewDocument` before navigation and `removeScriptToEvaluateOnNewDocument` after. Each navigation type catches its own error and appends a "Unable to navigate..." line rather than throwing.
- **Navigation allowlist (experimental):** `navigateWithInterception` enables `setRequestInterception(true)` when an `allowList` (comma-separated `URLPattern`s) is present; navigation requests not matching are `abort('blockedbyclient')`, non-navigation requests `continue()`. Interception is always torn down in `finally`.
- **`new_page` isolated context:** `isolatedContext` name → pages sharing the name share cookies/storage; different names are fully isolated.

## Relationships
- **Depends on:** [McpContext](../../03-context-state/mcp-context.md) (`getPageById`, `selectPage`, `newPage`, `closePage`), [ContextPage](../tool-definition.md) (`pptrPage`, `waitForEventsAfterAction`, `clearDialog`), Puppeteer/CDP (`URLPattern`, `CdpPage`).
- **Used by:** essentially every other tool, which act on the *selected* page set here.

## Gotchas & non-obvious details
- `resize_page` is categorized EMULATION even though it lives in `pages.ts` — gating follows the per-tool category.
- The last open page cannot be closed; `close_page` swallows `CLOSE_PAGE_ERROR` into a normal response line.
- `get_tab_id` reaches into the private `_tabId` and is gated behind `experimentalInteropTools`.
- `new_page`/`navigate_page` only add the `allowList` param when `experimentalNavigationAllowlist` is set (factory form).
- `select_page`'s `pageId` description programmatically references `list_pages().name`.

## Update triggers
- A tool is added/removed in `src/tools/pages.ts`, or its category changes.
- `navigateWithInterception` allowlist semantics change.
