# TextSnapshot & the UID system

> **Layer:** Context & state · **Sources:** `src/TextSnapshot.ts`, `src/types.ts`, `src/McpPage.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`TextSnapshot` captures the page's accessibility tree as a serializable node tree and assigns each node a stable UID. UIDs are how the LLM refers to elements: tools accept `{uid}` params and resolve them back to live `ElementHandle`s via `McpPage.getElementByUid`. The same snapshot also records which element is selected in the DevTools UI.

## Key files
- `src/TextSnapshot.ts` — the `TextSnapshot` class and `create()` factory.
- `src/types.ts` — `TextSnapshotNode extends SerializedAXNode` (adds `id`, `backendNodeId`, `loaderId`, `children`).
- `src/McpPage.ts` — owns the current `textSnapshot` and the `uniqueBackendNodeIdToMcpId` UID map.

## Key types & functions
- `TextSnapshot` — holds `root`, `idToNode: Map<string, TextSnapshotNode>`, `snapshotId`, `selectedElementUid?`, `hasSelectedElement`, `verbose` (`src/TextSnapshot.ts:17`).
- `static create(page, options?)` — builds a snapshot from the AX tree (`:47`).
- `TextSnapshotNode` — a serialized AX node + `id` (the UID), optional `backendNodeId`/`loaderId`, `children` (`src/types.ts:15`).

## How UID assignment works
`create()` calls `page.pptrPage.accessibility.snapshot({includeIframes: true, interestingOnly: !verbose})`, then walks the tree with `assignIds`:
- Each node has a `uniqueBackendId = \`${loaderId}_${backendNodeId}\``.
- It looks that up in the page's persistent `uniqueBackendNodeIdToMcpId` map. If found, the node **reuses** the existing UID; otherwise it mints a new one as `\`${snapshotId}_${idCounter++}\`` and records the mapping.
- Every node is also indexed into `idToNode` by its UID.

So UIDs are **stable across snapshots for the same DOM node** (same loader + backend node), which lets the LLM keep referring to an element after intermediate snapshots. The `snapshotId` (a process-global counter, `TextSnapshot.nextSnapshotId`) only prefixes *newly minted* UIDs. After building, `create()` prunes `uniqueBackendNodeIdToMcpId` entries that weren't seen this pass.

## Selected-element detection
If `options.devtoolsData` (or `page.getDevToolsData()`) reports a `cdpBackendNodeId`, the snapshot sets `hasSelectedElement = true` and `selectedElementUid = page.resolveCdpElementId(cdpBackendNodeId)` — letting the snapshot mark the element currently selected in the DevTools Elements panel.

## Extra nodes (third-party / non-AX elements)
`insertExtraNodes` grafts DOM nodes that aren't in the accessibility tree (e.g. handles returned by third-party developer tools, stored in `page.extraHandles`) into the snapshot:
1. `createExtraNode` mints a node for each handle (skipping ones already seen), using the element's `localName` as its role and `custom_<backendNodeId>` as its unique key.
2. `findAncestorNode` walks up the DOM via `parentElement` until it finds a handle whose `backendNodeId` matches an existing snapshot node (or falls back to root).
3. `findDescendantNodes` uses CDP `DOM.describeNode` (depth -1, pierce) to find which existing children belong *under* the extra node; `moveChildNodes` re-parents them so the tree stays structurally correct.

## Special-case: option values
For `role === 'option'` nodes, the AX node lacks a `value`, so `assignIds` copies the option's `name` text into `value`.

## Relationships
- **Depends on:** [Browser & CDP](../04-browser-cdp/README.md) (Puppeteer `accessibility.snapshot`, `ElementHandle`, CDP `DOM.describeNode`), [McpPage](mcp-page.md) (UID map + extra handles + DevTools data).
- **Used by:** [McpPage](mcp-page.md) (`getElementByUid`, `resolveCdpElementId`, `getAXNodeByUid`), [McpResponse](mcp-response.md) (rebuilds it in `handle()` then formats via `SnapshotFormatter`), [Formatters](../05-formatters/README.md) (`SnapshotFormatter` serializes `TextSnapshotNode`s).

## Gotchas & non-obvious details
- UID format `snapshotId_idx` is only the *minting* format; a reused node keeps its original UID even in later snapshots with a higher `snapshotId`.
- UID stability lives in `McpPage.uniqueBackendNodeIdToMcpId`, not in the `TextSnapshot` object — a snapshot is otherwise immutable per capture.
- `verbose: false` uses `interestingOnly` (the default), pruning uninteresting AX nodes; `verbose: true` includes everything.
- Extra-node insertion mutates the freshly built tree and is the reason `McpPage.extraHandles` exists.
- `resolveCdpElementId` (in `McpPage`) does a BFS to map a backend node ID back to a UID — there's a TODO to index by `backendNodeId` instead.

## Update triggers
- Change to UID format/stability, the AX-tree capture options, selected-element detection, or extra-node insertion logic.
