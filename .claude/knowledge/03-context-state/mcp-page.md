# McpPage

> **Layer:** Context & state · **Sources:** `src/McpPage.ts`, `src/tools/ToolDefinition.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`McpPage` is the per-page state wrapper around a Puppeteer `Page`. It consolidates the dialog, current text snapshot, emulation settings, and metadata that used to be scattered across `Map`s in `McpContext`. It implements the `ContextPage` interface that page-scoped tools receive.

## Key files
- `src/McpPage.ts` — the class.
- `src/tools/ToolDefinition.ts` — the `ContextPage` interface (`:279`).

## Key types & functions
- `McpPage implements ContextPage` (`src/McpPage.ts:42`).
- `getElementByUid(uid)` — resolves a snapshot UID to a live `ElementHandle` (`:337`).
- `resolveCdpElementId(backendNodeId)` — reverse lookup: CDP backend node ID → snapshot UID (`:372`).
- `waitForEventsAfterAction(action, options?)` — runs an action and waits for navigation/DOM-stable via `WaitForHelper` (`:133`).
- `executeThirdPartyDeveloperTool(toolName, params, response)` — bridges in-page WebMCP/third-party tools (`:148`).

## State it owns
- `readonly pptrPage: Page`, `readonly id: number` — the wrapped page and its LLM-facing ID.
- `textSnapshot: TextSnapshot | null` — the most recent accessibility snapshot. See [text-snapshot.md](text-snapshot.md).
- `uniqueBackendNodeIdToMcpId: Map<string, string>` — persists UID assignments across snapshots so the same DOM node keeps the same UID.
- `extraHandles: ElementHandle[]` — extra DOM nodes (from third-party tools) injected into the snapshot tree.
- `emulationSettings: EmulationSettings` — backing store for the `networkConditions`/`cpuThrottlingRate`/`geolocation`/`viewport`/`userAgent`/`colorScheme`/`extraHttpHeaders` getters.
- `isolatedContextName?`, `devToolsPage?` — metadata (incognito context name; the associated DevTools UI page).
- `thirdPartyDeveloperTools: ToolGroups` — cached tool groups discovered on the page.
- `#dialog?: Dialog` + `#dialogHandler` — current open dialog, captured by a `'dialog'` listener.

## How it works
- **Dialogs.** The constructor registers a `'dialog'` listener that stashes the dialog in `#dialog`. `getDialog()`/`throwIfDialogOpen()`/`clearDialog()` expose it; `dispose()` removes the listener. Tools with `blockedByDialog` call `throwIfDialogOpen()` before running.
- **Emulation getters** read from `emulationSettings` with sensible defaults (e.g. `cpuThrottlingRate` defaults to `1`). `McpContext.emulate` is what writes `emulationSettings`.
- **UID → element.** `getElementByUid` looks up the node in `textSnapshot.idToNode`, then `#resolveElementHandle` calls `node.elementHandle()`, throwing a clear "no longer exists" error if the node is stale.
- **Waiting.** `createWaitForHelper(cpuMult, networkMult)` (public for test spying) builds a `WaitForHelper`; `waitForEventsAfterAction` derives the multipliers from `cpuThrottlingRate` and `getNetworkMultiplierFromString(networkConditions)`. See [concurrency.md](concurrency.md).

## Third-party / WebMCP tool execution
`executeThirdPartyDeveloperTool` is the most involved method. It (1) resolves any `{uid}` params to `ElementHandle`s, (2) `evaluate`s the in-page tool via `window.__dtmcp.executeTool`, recursively sanitizing the result (DOM elements → stashed IDs, non-plain objects → `<Ctor instance>`, circular refs → `<Circular reference>`, functions → `<Function object>`), (3) pulls stashed DOM elements back out as `ElementHandle`s, (4) if any were returned, rebuilds the `textSnapshot` with them as `extraHandles` and calls `response.includeSnapshot()`, then (5) replaces stashed IDs in the result with `{uid}` references and appends the JSON to the response.

## Relationships
- **Depends on:** [TextSnapshot](text-snapshot.md), [WaitForHelper](concurrency.md), [Browser & CDP](../04-browser-cdp/README.md) (Puppeteer `Page`, `ElementHandle`, `Dialog`).
- **Used by:** [McpContext](mcp-context.md) (owns the `Page → McpPage` map), [Tool system](../02-tools/README.md) (`request.page`), [McpResponse](mcp-response.md) (reads emulation/dialog/snapshot for serialization).

## Gotchas & non-obvious details
- Fields are intentionally public for direct read/write by `McpContext`; only `#dialog` is private because it has listener lifecycle.
- `uniqueBackendNodeIdToMcpId` is the key to UID stability — it survives across `TextSnapshot.create` calls and is pruned of unseen IDs each snapshot.
- `getDevToolsData()` reaches into the DevTools UI page (if any) to read the currently selected network request / DOM node, enabling "selected in DevTools" cross-references.
- `resolveCdpElementId` does a BFS over the snapshot tree (TODO in source: index by `backendNodeId`).

## Update triggers
- New per-page state field, dialog-handling change, UID-resolution change, or change to third-party/WebMCP tool execution.
