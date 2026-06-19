# Browser & CDP integration

> **Layer:** Browser & CDP integration · **Sources:** `src/browser.ts`, `src/devtools/DevToolsConnectionAdapter.ts`, `src/devtools/McpHostBindingAdapter.ts`, `src/devtools/DevtoolsUtils.ts`, `src/third_party/index.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
This layer is how the server obtains a Chrome instance (launch a fresh one, or connect to a running one) and how it bridges Puppeteer/CDP into the **vendored `chrome-devtools-frontend` SDK** so the server can reuse DevTools' rich models (stack-trace symbolization, source-map resolution, issues, etc.) rather than reimplementing them.

## Key files
- `src/browser.ts` — obtain a `Browser`: launch vs. connect, channel mapping, Linux DISPLAY detection, target filtering, shutdown.
- `src/devtools/DevToolsConnectionAdapter.ts` — makes a Puppeteer `CDPSession`/`Connection` look like a DevTools `CDPConnection`, forwarding raw CDP events.
- `src/devtools/McpHostBindingAdapter.ts` — implements the DevTools frontend `InspectorFrontendHostAPI` (mostly no-ops) so the SDK can run host-less inside Node.
- `src/devtools/DevtoolsUtils.ts` — installs DevTools globals, the `UniverseManager` (one DevTools "universe"/target per page), and `SymbolizedError`/stack-trace helpers.
- `src/third_party/index.ts` — re-exports `puppeteer` (puppeteer-core) and `DevTools` (the vendored `chrome-devtools-frontend/mcp/mcp.js`).

## Key types & functions
- `ensureBrowserConnected` / `ensureBrowserLaunched` (`src/browser.ts`) — idempotent getters that cache a module-level singleton `Browser`.
- `overrideDevToolsGlobals` (`DevtoolsUtils.ts`) — one-time global bootstrap of the vendored DevTools SDK.
- `UniverseManager` (`DevtoolsUtils.ts`) — per-page DevTools target/universe lifecycle, keyed off Puppeteer `targetcreated`/`targetdestroyed`.
- `PuppeteerDevToolsConnection` (`DevToolsConnectionAdapter.ts`) — the `CDPConnection` shim.
- `McpHostBindingAdapter` (`McpHostBindingAdapter.ts`) — the `InspectorFrontendHostAPI` shim.

## How it works
1. **Get Chrome** (`src/index.ts:109`): if `--browser-url`, `--ws-endpoint`, or `--auto-connect` is set, the server calls `ensureBrowserConnected`; otherwise `ensureBrowserLaunched`. Both return a cached Puppeteer `Browser`.
2. **Build context** ([McpContext](../03-context-state/mcp-context.md) `from`): the constructor calls `overrideDevToolsGlobals(...)` exactly once to install the host bindings and stub fragile DevTools agents, then creates a `UniverseManager` over the `Browser`.
3. **Per-page DevTools universe**: `UniverseManager.init(pages)` creates a `TargetUniverse` per page — a secondary `CDPSession`, a `PuppeteerDevToolsConnection`, and a DevTools `Target` registered in a fresh `Universe`. New tabs are picked up automatically via browser target events.
4. **Consumers** (e.g. [ConsoleFormatter](../05-formatters/console-formatter.md) via `McpContext.getDevToolsUniverse(page)`) use the universe's DevTools models to symbolize errors and resolve source-mapped stack traces.

## Index of files below
- [browser-launch.md](browser-launch.md) — `src/browser.ts`
- [connection-adapters.md](connection-adapters.md) — `PuppeteerDevToolsConnection`
- [host-binding-adapter.md](host-binding-adapter.md) — `McpHostBindingAdapter` (vendored-frontend hub)
- [devtools-utils.md](devtools-utils.md) — globals override, `UniverseManager`, `SymbolizedError`

## Relationships
- **Depends on:** `puppeteer-core` and the vendored `chrome-devtools-frontend` SDK (both re-exported via `src/third_party/index.ts`).
- **Used by:** [McpContext](../03-context-state/mcp-context.md) (creates the `UniverseManager`, calls `overrideDevToolsGlobals`); `src/index.ts` (chooses launch vs. connect); `src/bin/chrome-devtools-mcp-main.ts` (`closeBrowser` on shutdown); [PageCollector](../03-context-state/collectors.md) (`FakeIssuesManager`); [ConsoleFormatter](../05-formatters/console-formatter.md) (`SymbolizedError`, `createStackTrace*`).

## Gotchas & non-obvious details
- The `Browser` is a **module-level singleton** in `src/browser.ts`, not stored on the context. There is at most one Chrome per process.
- The entire DevTools side is **coupling to vendored frontend internals** (`chrome-devtools-frontend@1.0.1646286`); much of `DevtoolsUtils.ts` reaches into agent prototypes and private models. See per-file Gotchas.

## Update triggers
- `chrome-devtools-frontend` or `puppeteer-core` dependency is bumped.
- A new connection mode (beyond launch / browser-url / ws-endpoint / auto-connect) is added.
- The DevTools universe wiring (per-page target model) changes.
