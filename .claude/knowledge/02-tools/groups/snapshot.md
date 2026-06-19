# Tool Group: Snapshot

> **Layer:** Tool system · **Sources:** `src/tools/snapshot.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Tools to capture a text (accessibility-tree) snapshot of the selected page — the canonical way agents discover element `uid`s — and to wait for specific text to appear before snapshotting.

## Key files
- `src/tools/snapshot.ts` — `take_snapshot`, `wait_for`.

## Key types & functions
Both are `definePageTool`.

- `take_snapshot` — category **DEBUGGING**, `readOnlyHint: false` (because of `filePath`), `blockedByDialog: true`. Schema `{verbose?, filePath?}`; `verifyFilesSchema: ['filePath']`. Calls `response.includeSnapshot({verbose: verbose ?? false, filePath})`.
- `wait_for` — category **NAVIGATION**, `readOnlyHint: true`, `blockedByDialog: true`. Schema `{text: string[] (min 1), timeout?}`. Calls `context.waitForTextOnPage(text, timeout, page.pptrPage)`, appends a "found" line, then `response.includeSnapshot()`.

## How it works
- The snapshot is an accessibility-tree text rendering listing elements with stable `uid`s. The handler only flags `response.includeSnapshot(...)`; the actual a11y tree capture and uid assignment happen in the response/context layer (`TextSnapshot`, `McpPage.getAXNodeByUid`/`getElementByUid`). The snapshot also marks the element selected in the DevTools Elements panel, per the tool description.
- `verbose: true` includes the full a11y tree; default is the trimmed view.
- `wait_for` resolves as soon as *any* of the provided strings appears (`text` is a non-empty array, OR semantics), using `context.waitForTextOnPage`, then attaches a fresh snapshot so the agent immediately gets the new uids.
- `timeout` uses the shared `timeoutSchema` (`<= 0` coerced to undefined → default timeout).

## Relationships
- **Depends on:** [McpResponse](../../03-context-state/mcp-response.md) (`includeSnapshot`), [McpContext](../../03-context-state/mcp-context.md) (`waitForTextOnPage`), [ContextPage](../tool-definition.md) (`pptrPage`), `TextSnapshot` (`src/TextSnapshot.ts`).
- **Used by:** every uid-based [input](input.md) tool, [screenshot](screenshot.md) (`uid`), and [script](script.md) (`args` element uids).

## Gotchas & non-obvious details
- `take_snapshot` is `readOnlyHint: false` solely because the optional `filePath` can write the snapshot to disk.
- `wait_for` is NAVIGATION category even though it reads the page — it's about waiting for navigation/render to settle on expected text.
- The tool description tells the model to prefer a snapshot over a screenshot and to always use the latest snapshot (uids from stale snapshots may not resolve).

## Update triggers
- A tool is added in `src/tools/snapshot.ts`.
- The `includeSnapshot` params or `waitForTextOnPage` signature change.
