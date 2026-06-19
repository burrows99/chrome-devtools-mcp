# Entrypoints & process model

> **Layer:** Entrypoints & process model · **Sources:** `src/bin/*`, `src/index.ts`, `src/daemon/*`, `src/version.ts`, `src/polyfill.ts`, `src/utils/check-for-updates.ts`, `server.json`, `package.json` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
This layer is everything that turns the package into a running process: the two published binaries (`chrome-devtools-mcp` and `chrome-devtools`), the MCP server bootstrap (`src/index.ts`), and the background daemon that backs the synchronous CLI. It owns argument parsing, Node version gating, update checks, signal/shutdown handling, and transport setup.

## Key files
- `src/bin/chrome-devtools-mcp.ts` — `bin` shim: sets `process.title`, enforces Node version, then dynamically imports the real main.
- `src/bin/chrome-devtools-mcp-main.ts` — the MCP server process: wires shutdown handlers, creates the server, connects a stdio transport.
- `src/bin/chrome-devtools-mcp-cli-options.ts` — `cliOptions` (yargs option spec) + `parseArguments()`, shared by the server and the daemon.
- `src/bin/chrome-devtools.ts` — `bin` shim + yargs command tree for the synchronous `chrome-devtools` CLI (start/stop/status + one subcommand per tool).
- `src/bin/chrome-devtools-cli-options.ts` — auto-generated `commands` map (one entry per MCP tool) consumed by the CLI.
- `src/bin/check-latest-version.ts` — detached helper process that fetches the latest npm version into a cache file.
- `src/index.ts` — `createMcpServer()` / `logDisclaimers()`; builds the `McpServer`, registers tools, lazily launches/connects Chrome.
- `src/daemon/daemon.ts` — long-lived background process: owns the PID file, listens on a socket, proxies to an MCP child over stdio.
- `src/daemon/client.ts` — spawns/stops the daemon, `sendCommand()` over the socket, `handleResponse()` formatting.
- `src/daemon/utils.ts` — socket/PID path resolution, `isDaemonRunning()`, `serializeArgs()`.
- `src/daemon/types.ts` — `DaemonMessage` / `DaemonResponse` IPC shapes.
- `src/version.ts` — single `VERSION` constant (release-please managed).
- `src/polyfill.ts` — re-export of bundled polyfills (imported first by the server main).
- `src/utils/check-for-updates.ts` — cached "update available" notifier; spawns `check-latest-version.js`.

## Key types & functions
- `createMcpServer(serverArgs, {logFile})` — builds and returns `{server}`; the server's transport is attached by the caller (`src/index.ts`).
- `parseArguments(version, argv?, env?)` — yargs parse of `cliOptions`; returns `ParsedArguments`.
- `startDaemon` / `stopDaemon` / `sendCommand` / `handleResponse` — daemon client API (`src/daemon/client.ts`).
- `isDaemonRunning(sessionId)` / `serializeArgs(options, argv)` — daemon liveness check & arg round-tripping (`src/daemon/utils.ts`).

## How it works
There are **three distinct ways a process starts** (see the per-file docs for detail):

1. **MCP server** (`chrome-devtools-mcp` bin → `chrome-devtools-mcp-main.ts`). This is the canonical MCP entry: a stdio MCP server that AI clients spawn. See [server-bootstrap.md](server-bootstrap.md) and [binaries.md](binaries.md).
2. **Synchronous CLI** (`chrome-devtools` bin → `chrome-devtools.ts`). A human-friendly command tree (`start`, `stop`, `status`, plus one subcommand per tool). Each tool subcommand auto-starts a daemon if needed, then issues one request. See [cli.md](cli.md).
3. **Daemon** (`src/daemon/daemon.ts`, spawned by the CLI client). A detached background process that runs *its own* `chrome-devtools-mcp` MCP server as a stdio child and exposes it over a local socket so repeated CLI invocations reuse one browser session. See [daemon.md](daemon.md).

Both binaries call `checkForUpdates(...)` first, share `parseArguments`/`cliOptions`, and share `logDisclaimers`. Only the MCP-server path attaches a `StdioServerTransport`; the daemon path runs the MCP server as a stdio *client* of that same binary.

## Relationships
- **Depends on:** the tool layer ([../02-tools/](../02-tools/) — `createTools`, `ToolHandler`), context/state ([../03-context-state/](../03-context-state/) — `McpContext`), browser/CDP ([../04-browser-cdp/](../04-browser-cdp/) — `ensureBrowserLaunched`/`ensureBrowserConnected`/`closeBrowser`), telemetry ([../07-telemetry/](../07-telemetry/) — `ClearcutLogger`), and vendored third-party re-exports (`src/third_party/index.ts`).
- **Used by:** MCP clients (over stdio) and humans (the CLI). The package `bin` map points the two published commands here.

## Gotchas & non-obvious details
- `VERSION` lives in `src/version.ts` and is mirrored in `package.json` and `server.json`; all three must match a release (release-please markers in `version.ts`).
- The `chrome-devtools-mcp.ts` shim does Node-version gating **before** importing anything heavy — keep that ordering.
- The daemon spawns its MCP server via `INDEX_SCRIPT_PATH = .../bin/chrome-devtools-mcp.js` (the bin shim, not `index.js`).

## Update triggers
- A new file appears under `src/bin/` or `src/daemon/`, or a binary is added/removed in `package.json` `bin`.
- `createMcpServer`'s signature, transport, or tool-registration flow changes.
- The daemon IPC protocol (`DaemonMessage`/`DaemonResponse`) or socket/PID path scheme changes.
