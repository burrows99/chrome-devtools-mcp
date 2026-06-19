# DevTools globals, UniverseManager & error symbolization

> **Layer:** Browser & CDP integration · **Sources:** `src/devtools/DevtoolsUtils.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Bootstrap the vendored DevTools SDK for headless use, then maintain one DevTools **universe** (an isolated SDK world with its own settings + target manager) per Puppeteer page, and provide helpers that use those universes to symbolize errors and build source-mapped stack traces.

## Key files
- `src/devtools/DevtoolsUtils.ts` — `overrideDevToolsGlobals`, `UniverseManager` + `TargetUniverse`, `DEFAULT_FACTORY`, `SymbolizedError`, `createStackTrace*`, `FakeIssuesManager`.

## Key types & functions
- `overrideDevToolsGlobals({loadResource})` — one-time global SDK setup (host bindings, agent stubs, locale, formatter worker).
- `TargetUniverse` = `{target: DevTools.Target, universe: DevTools.Foundation.Universe.Universe, session: CDPSession}`.
- `UniverseManager` — creates/tracks/disposes a `TargetUniverse` per `Page`; `init(pages)`, `get(page)`, `dispose()`.
- `DEFAULT_FACTORY: TargetUniverseFactoryFn` — the per-page universe builder.
- `SymbolizedError` — message + resolved stack trace + resolved `cause` chain, built from a `Runtime.ExceptionDetails` or `RemoteObject`.
- `createStackTrace` / `createStackTraceForConsoleMessage` — resolve raw protocol stack traces (waiting for source maps) into DevTools `StackTrace`.
- `FakeIssuesManager` — minimal `IssuesManager` stub (`issues()` → `[]`) for the issues aggregator.

## How it works
**`overrideDevToolsGlobals`** runs once (from `McpContext`'s constructor) and:
- installs the `McpHostBindingAdapter` via `installInspectorFrontendHost` (see [host-binding-adapter.md](host-binding-adapter.md));
- sets `InspectorBackend.test.suppressRequestErrors = true` (CDP errors get noisy);
- **monkey-patches the `Network` agent prototype** so several `invoke_*` methods (`emulateNetworkConditionsByRule`, `overrideNetworkState`, `enable`, `disable`, `setBlockedURLs`) become no-ops returning a fake success — this stops the DevTools frontend from ever clearing Puppeteer's active blocking/throttling rules during target setup;
- forces a fixed `en-US` `DevToolsLocale` and registers empty locale data;
- instantiates the `FormatterWorkerPool` pointing at the bundled `../third_party/devtools-formatter-worker.js`.

**`UniverseManager`** is constructed with a `Browser` and (by default) `DEFAULT_FACTORY`, holding universes in a `WeakMap<Page, TargetUniverse>` guarded by a `Mutex`. `init(pages)` builds a universe per existing page and subscribes to the browser's `targetcreated`/`targetdestroyed` to add/remove universes for new tabs; `dispose()` unsubscribes.

**`DEFAULT_FACTORY`** per page: builds a `Universe` with in-memory `SettingsStorage` and `overrideAutoStartModels: new Set([DebuggerModel])`; opens a **secondary** `page.createCDPSession()`, wraps it in a `PuppeteerDevToolsConnection` (see [connection-adapters.md](connection-adapters.md)); gets the `TargetManager` and observes models — `DebuggerModel` with `SKIP_ALL_PAUSES` (calls `invoke_setSkipAllPauses({skip:true})`) and `NetworkManager` with `DISABLE_NETWORK` (calls `networkAgent().invoke_disable()`); then `targetManager.createTarget('main', '', 'frame', null, session.id(), undefined, connection)`.

**`SymbolizedError`** (`fromDetails`/`fromError`) parses the message, optionally resolves the stack trace via `createStackTrace`, and follows the error's `cause` property (via `Runtime.getProperties`) into a recursive `SymbolizedError`. `createStackTrace` collects all script ids up front and waits (≤1s `AbortSignal.timeout`) for pending source maps before calling `DebuggerWorkspaceBinding.createStackTraceFromProtocolRuntime`, because DevTools' normal "retranslate after source map attaches" event flow doesn't apply in the MCP context.

## Relationships
- **Depends on:** the vendored **`chrome-devtools-frontend`** SDK (`Foundation.Universe`, `TargetManager`, `DebuggerModel`, `NetworkManager`, `DebuggerWorkspaceBinding`, `Settings`, `I18n`, `FormatterWorkerPool`); [connection-adapters.md](connection-adapters.md); [host-binding-adapter.md](host-binding-adapter.md); `src/Mutex.ts`.
- **Used by:** [McpContext](../03-context-state/mcp-context.md) (`overrideDevToolsGlobals`, `UniverseManager`, `getDevToolsUniverse`); [ConsoleFormatter](../05-formatters/console-formatter.md) (`SymbolizedError`, `createStackTrace*`); [PageCollector](../03-context-state/collectors.md) (`FakeIssuesManager`).

## Gotchas & non-obvious details
- **Heavy vendored-frontend coupling (inferred internals):** the `Network` agent prototype patch reaches into `InspectorBackend.inspectorBackend.agentPrototypes.get('Network')` and redefines `invoke_*` props; a frontend bump could rename these and silently re-enable resets of Puppeteer's network rules.
- Each page uses a **secondary** CDP session (distinct from the one Puppeteer drives the page with) so DevTools models don't fight Puppeteer for the primary session.
- `setSkipAllPauses` only suppresses pause/resume events *on the MCP session* — DevTools may still pause/step elsewhere; MCP just won't observe `Debugger.paused`/`resumed`.
- `'frame' as any` and several `as`-casts in `createTarget`/stack-trace code paper over DevTools' branded protocol types vs. Puppeteer's `Protocol` types (asserted safe in comments).
- `waitForScript` loops indefinitely and is intentionally paired with the 1s abort signal; without source maps resolving in time the stack falls back unsymbolized.

## Update triggers
- `chrome-devtools-frontend` dependency is bumped (agent prototype names, `Universe`/`TargetManager`/`createTarget` signatures, model classes).
- The per-page universe model changes (e.g. enabling the DevTools network domain, recording pauses).
- The bundled `devtools-formatter-worker.js` path/build changes.
