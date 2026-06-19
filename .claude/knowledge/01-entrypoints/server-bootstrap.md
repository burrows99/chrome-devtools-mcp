# Server bootstrap (`src/index.ts`)

> **Layer:** Entrypoints & process model · **Sources:** `src/index.ts`, `src/version.ts`, `src/polyfill.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`createMcpServer()` builds the `McpServer`, registers every enabled tool, wires MCP lifecycle handlers (roots, logging, client identification), and lazily creates the Chrome-backed `McpContext`. `logDisclaimers()` prints the standard privacy/telemetry notices to stderr.

## Key files
- `src/index.ts` — `createMcpServer()`, `logDisclaimers()`, re-exports `buildFlag`.
- `src/version.ts` — `VERSION` constant baked into the server's identity.
- `src/polyfill.ts` — bundled polyfills (imported by the bin main before this module).

## Key types & functions
- `createMcpServer(serverArgs: ParsedArguments, options: {logFile?: fs.WriteStream}) => Promise<{server}>` — main bootstrap. The caller attaches the transport.
- `logDisclaimers(args)` — stderr notices (always the data-exposure warning; CrUX & usage-stats notices unless `slim`/disabled).
- `buildFlag(category)` — re-exported from `ToolHandler`; maps a tool category to its `category*` CLI flag.

## How it works

### Telemetry init
If `serverArgs.usageStatistics`, `ClearcutLogger.initialize(...)` is called with a `FilePersistence`, the log file, `VERSION`, and the hidden `clearcut*` test knobs.

### Server construction
```ts
const server = new McpServer(
  {name: 'chrome_devtools', title: 'Chrome DevTools MCP server', version: VERSION},
  {capabilities: {logging: {}}},
);
```
It registers a no-op `SetLevelRequestSchema` handler (accepts client log-level changes). On `server.server.oninitialized` it records the client name into `ClearcutLogger` and, if the client advertises `roots`, calls `updateRoots()` and subscribes to `RootsListChangedNotificationSchema`.

### Lazy context (`getContext`)
`context` (an `McpContext`) is **not** created at startup — it is created on first tool call via the `getContext` closure passed into each `ToolHandler`. `getContext`:
- Assembles `chromeArgs` (incl. `--proxy-server=` from `proxyServer`), `ignoreDefaultChromeArgs`, blockedlist/allowlist from `blockedUrlPattern`/`allowedUrlPattern`.
- Branches on connection mode: if `browserUrl || wsEndpoint || autoConnect` → `ensureBrowserConnected(...)` (channel only passed when `autoConnect`); otherwise → `ensureBrowserLaunched(...)` (headless, executablePath, channel, isolated, userDataDir, viewport, chromeArgs, extensions, etc.).
- Rebuilds `McpContext.from(browser, logger, {...})` only when `context?.browser !== browser`, then re-runs `updateRoots()`.
This means **Chrome is launched/connected on demand**, not when the MCP server connects.

### Tool registration
```ts
const tools = createTools(serverArgs);
for (const tool of tools) registerTool(tool);
await loadIssueDescriptions();
```
`registerTool` wraps each tool in a `ToolHandler(tool, serverArgs, getContext, toolMutex)` (a single shared `Mutex` serializes tool execution). If `toolHandler.shouldRegister` is false, the tool is skipped; otherwise `server.registerTool(name, {description, inputSchema: registeredInputSchema, annotations}, handler)`. `shouldRegister` is `!(disabled && !serverArgs.viaCli)` — i.e. disabled tools are **still registered when launched via the CLI** so the CLI can surface a helpful "enable with `category*=true`" error instead of "unknown command".

## Relationships
- **Depends on:** tools ([../02-tools/](../02-tools/) — `createTools`, `ToolHandler`, `ToolDefinition`), context ([../03-context-state/](../03-context-state/) — `McpContext`, `Mutex`), browser ([../04-browser-cdp/](../04-browser-cdp/) — `ensureBrowserConnected`/`ensureBrowserLaunched`), telemetry ([../07-telemetry/](../07-telemetry/) — `ClearcutLogger`, `FilePersistence`), `issue-descriptions`, and `src/third_party/index.ts` (`McpServer`, schemas).
- **Used by:** `chrome-devtools-mcp-main.ts` (server path) and `chrome-devtools.ts` (CLI calls `logDisclaimers` directly; the daemon runs the bin which calls `createMcpServer`).

## Gotchas & non-obvious details
- `createMcpServer` returns `{server}` **without** a transport — the caller decides stdio vs. (client-side) wiring.
- Context creation is lazy and memoized; the first tool call pays the browser-launch latency.
- A single `toolMutex` serializes *all* tool invocations on a server instance — tools do not run concurrently.
- `loadIssueDescriptions()` is awaited at the end of bootstrap; it must complete before tools that surface issue text run.

## Update triggers
- The `McpServer` capabilities, name/title, or lifecycle handlers change.
- The connection-mode branching in `getContext` changes (new connection flag, channel handling).
- `shouldRegister` / `viaCli` registration semantics change in `ToolHandler`.
