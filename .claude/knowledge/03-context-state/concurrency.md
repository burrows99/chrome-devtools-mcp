# Concurrency: Mutex & WaitForHelper

> **Layer:** Context & state · **Sources:** `src/Mutex.ts`, `src/WaitForHelper.ts`, `src/ToolHandler.ts`, `src/McpPage.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Two helpers govern timing. `Mutex` serializes tool execution so only one tool mutates browser state at a time. `WaitForHelper` runs an action (a click, fill, navigation, etc.) and then waits for any resulting navigation and for the DOM to settle before the tool returns — so the next snapshot reflects the post-action page.

## Key files
- `src/Mutex.ts` — a tiny FIFO async mutex with a disposable `Guard`.
- `src/WaitForHelper.ts` — `WaitForHelper`, `WaitForEventsResult`, `getNetworkMultiplierFromString`.

## Key types & functions
- `Mutex.acquire(): Promise<Guard>` / `Mutex.release()` — FIFO lock (`src/Mutex.ts:22`).
- `Mutex.Guard.dispose()` — releases the lock; pairs with `using`/`finally` (`src/Mutex.ts:8`).
- `WaitForHelper.waitForEventsAfterAction(action, options?)` — the main API (`src/WaitForHelper.ts:131`).
- `getNetworkMultiplierFromString(condition)` — maps a Puppeteer network condition to a timeout multiplier (`:212`).

## Mutex
A minimal mutex: `#locked` boolean + a FIFO `#acquirers` queue of resolvers. `acquire()` returns immediately with a `Guard` if unlocked, otherwise queues a resolver and awaits it. `release()` (called by `Guard.dispose()`) wakes the next waiter or clears the lock.

Usage: a single `toolMutex = new Mutex()` (`src/index.ts:157`) is shared by all `ToolHandler`s. `ToolHandler.handle` does `const guard = await this.toolMutex.acquire()` and disposes it in `finally` — guaranteeing tools run strictly one-at-a-time even though the MCP server may receive concurrent requests. (The same `Mutex` class is also reused inside `UniverseManager` in `devtools/DevtoolsUtils.ts`.)

## WaitForHelper
Constructed per action via `McpPage.createWaitForHelper(cpuMult, networkMult)`. Timeouts scale with throttling: `stableDomTimeout = 3000 * cpuMult`, `stableDomFor = 100 * cpuMult`, `expectNavigationIn = 100 * cpuMult`, `navigationTimeout = 3000 * networkMult`.

`waitForEventsAfterAction(action, options)` orchestration:
1. If `options.handleDialog` is set, register a temporary `'dialog'` listener that auto-accepts/dismisses (or accepts with prompt text) and flags `#dialogOpened`.
2. Start a `navigationFinished` promise: `waitForNavigationStarted()` listens for CDP `Page.frameStartedNavigating`; a "real" (cross-document) navigation triggers `page.waitForNavigation(...)`, while history/same-document navigations resolve as "no navigation". If nothing starts within `expectNavigationIn`, it resolves false.
3. Run `action()`. On throw, abort all pending waits and rethrow.
4. Await `navigationFinished`. If a dialog opened, return immediately (skip DOM-stable wait). Otherwise `waitForStableDom()`.
5. Always `abort()` in `finally`; return `WaitForEventsResult` (`{navigatedToUrl}` if the URL changed).

`waitForStableDom()` injects a `MutationObserver` on `document.body` (childList/subtree/attributes) that resolves once `stableDomFor` ms pass with no mutation, racing a hard `stableDomTimeout`. An `AbortController` tears down the in-page observer on abort.

`McpPage.waitForEventsAfterAction` is the public wrapper that derives the multipliers from the page's `cpuThrottlingRate` and `getNetworkMultiplierFromString(networkConditions)`.

## Network multipliers
`getNetworkMultiplierFromString`: `Fast 4G`→1, `Slow 4G`→2.5, `Fast 3G`→5, `Slow 3G`→10, else 1. Used both here and by `McpContext.#updateSelectedPageTimeouts` to lengthen navigation timeouts under throttling.

## Relationships
- **Depends on:** [Browser & CDP](../04-browser-cdp/README.md) (Puppeteer `Page`, CDP `Page.frameStartedNavigating`, `Dialog`, `PredefinedNetworkConditions`).
- **Used by:** [Entrypoints](../01-entrypoints/README.md) (`ToolHandler` uses the `Mutex`), [McpPage](mcp-page.md) (`createWaitForHelper`/`waitForEventsAfterAction`), [Tool system](../02-tools/README.md) (action tools wrap mutations in `page.waitForEventsAfterAction`).

## Gotchas & non-obvious details
- A `WaitForHelper` is **single-use** — after its `AbortController` aborts, reuse throws `"Can't re-use a WaitForHelper"`. Create a fresh one per action.
- The DOM-stable wait is deliberately skipped when a dialog opened (the page is blocked and won't settle).
- History / same-document navigations are treated as "no navigation" so they don't trigger a full `waitForNavigation` timeout.
- `Mutex` is FIFO, so tool requests are served in arrival order; there's no fairness/priority beyond that.

## Update triggers
- Change to the tool-serialization model, the wait-for-action orchestration (navigation/DOM-stable logic), throttling timeout multipliers, or dialog auto-handling.
