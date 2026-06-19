# Tool Group: Script

> **Layer:** Tool system · **Sources:** `src/tools/script.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
A single tool, `evaluate_script`, that runs a JavaScript *function declaration* inside the selected page (or a frame, or — with extensions — a service worker) and returns the JSON-serialized result.

## Key files
- `src/tools/script.ts` — `evaluate_script` plus `performEvaluation`, `getPageOrFrame`, `getWebWorker` helpers.

## Key types & functions
- `evaluateScript` (MCP name `evaluate_script`, `defineTool` factory — **not** page-scoped) — category **DEBUGGING**, `readOnlyHint: false`, `blockedByDialog: true`. `verifyFilesSchema: ['filePath']`.
- Schema (conditional on CLI args): `{pageId?(experimentalPageIdRouting), function, args?, filePath?, dialogAction?, serviceWorkerId?(categoryExtensions)}`.
  - `function` — a JS function declaration string, e.g. `() => document.title` or `(el) => el.innerText`.
  - `args` — optional array of element `uid` strings passed as handles to the function.
  - `dialogAction` — `"accept"`/`"dismiss"`/prompt response string; default `accept`.
- `type Evaluatable = Page | Frame | WebWorker`.

## How it works
- **Target selection:** `defineTool` (non-page) here, so it resolves its own page: `context.getPageById(pageId)` under `experimentalPageIdRouting`, else `context.getSelectedMcpPage()`. With `serviceWorkerId` (extensions on) it evaluates in a `WebWorker` instead.
- **Element args:** each `uid` in `args` is resolved to an `ElementHandle` via `mcpPage.getElementByUid`; their `frame`s are collected. `getPageOrFrame` throws if uids span more than one frame ("Elements from different frames can't be evaluated together"), otherwise picks the single frame or the page.
- **Execution:** wrapped in `mcpPage.waitForEventsAfterAction(..., {handleDialog: dialogAction ?? 'accept'})` then `response.attachWaitForResult`. `performEvaluation` does `evaluateHandle('(' + fnString + ')')` then `evaluate((fn, ...args) => JSON.stringify(await fn(...args)), fn, ...args)`.
- **Output:** if `filePath` → `context.saveFile(data, filePath, '.json')` and report the filename; else append a fenced ```json block with the result. All handles `dispose()`d.
- **Service-worker path:** `getWebWorker` finds the worker by `serviceWorkerId`; rejects if `args` (uids) or `pageId` are also supplied.

## Relationships
- **Depends on:** [McpContext](../../03-context-state/mcp-context.md) (`getSelectedMcpPage`, `getPageById`, `getExtensionServiceWorkers`, `saveFile`), [McpPage](../../03-context-state/mcp-page.md) (`getElementByUid`, `waitForEventsAfterAction`), Puppeteer `Page`/`Frame`/`WebWorker`/`JSHandle`.
- **Used by:** agents running arbitrary page logic; `args` uids come from [snapshot](snapshot.md). Contrast the simpler slim `evaluate` ([slim mode](../slim-mode.md)).

## Gotchas & non-obvious details
- The result must be JSON-serializable — it is `JSON.stringify`'d inside the page; non-serializable returns are lost.
- `function` is a *declaration* (wrapped in parens and called), not a bare expression — different from the slim `evaluate` tool which runs a raw script string.
- Elements from different frames cannot be mixed in one call.
- It's a `defineTool` (not `definePageTool`), so it resolves the page itself and adds its own `pageId` param under page-id routing.
- Service-worker evaluation is mutually exclusive with `args`/`pageId`.

## Update triggers
- The schema, frame/worker resolution, or output routing changes.
- Conditional params (`pageId`, `serviceWorkerId`) gating changes.
