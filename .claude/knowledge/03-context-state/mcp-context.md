# McpContext

> **Layer:** Context & state · **Sources:** `src/McpContext.ts`, `src/tools/ToolDefinition.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`McpContext` is the central runtime-state hub. It owns the Puppeteer `Browser`, every page (`McpPage`), all data collectors, isolated browser contexts, emulation/screen-recording/trace state, heap-snapshot access, and extension management. It implements the read-only `Context` interface that tools depend on, making it the most-connected abstraction in the codebase.

## Key files
- `src/McpContext.ts` — the class (~985 lines).
- `src/tools/ToolDefinition.ts` — the `Context` and `ContextPage` interfaces tools see (a deliberately narrow subset).

## Key types & functions
- `McpContext implements Context` — the hub (`src/McpContext.ts:74`).
- `static from(browser, logger, opts, locatorClass?)` — the only public constructor; runs `#init()` (`:172`).
- `Context` — readonly interface; "Only add methods used by tools/*" (`src/tools/ToolDefinition.ts:180`).
- `McpContextOptions` — flags: `experimentalDevToolsDebugging`, `experimentalIncludeAllPages`, `performanceCrux`, `hasNetworkBlockOrAllowlist` (`src/McpContext.ts:60`).

## What it owns
Private fields (all `#`-prefixed):
- **Pages:** `#pages: Page[]`, `#mcpPages: Map<Page, McpPage>`, `#selectedPage?: McpPage`, `#nextPageId`.
- **Isolated contexts:** `#isolatedContexts: Map<string, BrowserContext>` (LLM-named incognito contexts) + `#nextIsolatedContextId`.
- **Collectors:** `#networkCollector` (`NetworkCollector`), `#consoleCollector` (`ConsoleCollector`), `#serviceWorkerConsoleCollector` (`ServiceWorkerConsoleCollector`). See [collectors.md](collectors.md).
- **DevTools:** `#devtoolsUniverseManager: UniverseManager` (secondary CDP session per page for DevTools UI integration).
- **Extensions:** `#extensionServiceWorkers`, `#extensionPages` (`WeakMap<Target, Page>`), `#extensionServiceWorkerMap` (`WeakMap<Target, string>`), `#nextExtensionServiceWorkerId`.
- **Heap snapshots:** `#heapSnapshotManager: HeapSnapshotManager`.
- **Recording/trace:** `#isRunningTrace`, `#screenRecorderData`, `#traceResults`.
- **Roots:** `#roots: Root[]` — MCP workspace roots used to sandbox file writes.

## Lifecycle
- **Create:** `McpContext.from(...)` → private ctor (overrides DevTools globals, instantiates collectors + `UniverseManager`) → `#init()` snapshots pages + extension SWs, then `init`s each collector and the universe manager.
- **Dispose:** `dispose()` disposes all collectors, the universe manager, every `McpPage`, clears the page map and isolated-context map. It does **not** close the browser or isolated contexts.

## Key methods (grouped)
- **Pages:** `newPage(background?, isolatedContextName?)`, `closePage(pageId)`, `selectPage`, `getSelectedPptrPage`, `getSelectedMcpPage`, `getPageById`, `getPageId`, `getPages`, `createPagesSnapshot()`. See [mcp-page.md](mcp-page.md).
- **Snapshot/UID:** `getAXNodeByUid(uid)` (linear scan over all pages' snapshots, `:564`). See [text-snapshot.md](text-snapshot.md).
- **Network:** `getNetworkRequests`, `getNetworkRequestById`, `getNetworkRequestStableId`, `resolveCdpRequestId`.
- **Console:** `getConsoleData`, `getConsoleMessageById`, `getConsoleMessageStableId`, `getServiceWorkerConsoleData`.
- **Emulation:** `emulate(options, targetPage?)` (network/CPU/geolocation/UA/color-scheme/viewport/headers), `restoreEmulation(page)`. Persists into `mcpPage.emulationSettings` and re-derives timeouts.
- **Files:** `validatePath`, `saveFile`, `saveTemporaryFile`, `loadResource` (http/https/file).
- **Heap:** `getHeapSnapshotAggregates/Stats/StaticData/NodesById/Retainers/RetainingPaths/Dominators/Edges`, `closeHeapSnapshot`, `hasHeapSnapshots`.
- **Extensions:** `installExtension`, `uninstallExtension`, `triggerExtensionAction`, `listExtensions`, `getExtension`, `getExtensionServiceWorkers`.
- **Trace/recording:** `setIsRunningPerformanceTrace`, `storeTraceRecording`, `recordedTraces`, `get/setScreenRecorder`, `isCruxEnabled`.
- **Waiting:** `waitForTextOnPage(text[], timeout?, page?)` races aria/text locators across all frames.

## How emulation works
`emulate()` is the single entry point for all per-page emulation. For each option it either applies the CDP/Puppeteer call and records the value into a fresh `EmulationSettings`, or clears it (when the option is falsy). It then writes `mcpPage.emulationSettings`, calls `#updateSelectedPageTimeouts()` (so throttling lengthens default/navigation timeouts), and finally `setViewport` last (viewport changes can trigger a reload). CPU throttling is mirrored onto the DevTools secondary session if present. Network throttling throws if a block/allowlist is configured (they conflict in Puppeteer).

## How page discovery works
`createPagesSnapshot()` is the reconciliation routine: `#getAllPages()` returns all Puppeteer pages plus extension page/side-panel targets (cached in `#extensionPages` because `target.asPage()` returns a new instance each call) and auto-discovered `BrowserContext`s mapped to isolated-context names. It then creates/prunes `McpPage` entries, enables focused-page emulation for every page (multi-agent support), filters out `devtools://` pages (unless `experimentalDevToolsDebugging`), re-selects a page if the current one vanished, and detects open DevTools windows.

## Relationships
- **Depends on:** [Browser & CDP](../04-browser-cdp/README.md), [Formatters](../05-formatters/README.md), [Performance & memory](../06-performance-memory/README.md) (`HeapSnapshotManager`, `TraceResult`), [Utilities](../08-utilities/README.md) (file helpers, `getNetworkMultiplierFromString`).
- **Used by:** [Tool system](../02-tools/README.md) (every handler receives it as `context`), [Entrypoints](../01-entrypoints/README.md) (`ToolHandler`, `src/index.ts`).
- **Internal:** [McpPage](mcp-page.md), [Collectors](collectors.md), [TextSnapshot](text-snapshot.md), [McpResponse](mcp-response.md).

## Gotchas & non-obvious details
- Never `new McpContext(...)`; use `McpContext.from` so collectors/universe manager are initialized.
- `getAXNodeByUid` scans every page's snapshot — fine because page counts are small (2–10) and a reverse index would complicate UID-reuse lifecycle.
- `roots()` always appends a synthetic `temp` root (`os.tmpdir()`) so temp files pass `validatePath`. If no roots are configured at all, `validatePath` is a no-op (allows everything).
- `#updateSelectedPageTimeouts()` multiplies `DEFAULT_TIMEOUT` (5s) by the CPU multiplier and `NAVIGATION_TIMEOUT` (10s) by both CPU and network multipliers.
- Isolated contexts created externally (incognito) are auto-named `isolated-context-N`.
- `setUpNetworkCollectorForTesting()` swaps in a collector that ignores favicon requests (test flakiness).

## Update triggers
- New owned subsystem/collector, new `Context` interface method, or a change to emulation/page-discovery/file-sandboxing logic.
