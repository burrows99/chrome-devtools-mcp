# The background daemon

> **Layer:** Entrypoints & process model · **Sources:** `src/daemon/daemon.ts`, `src/daemon/client.ts`, `src/daemon/utils.ts`, `src/daemon/types.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The daemon is a detached, long-lived process that keeps **one** Chrome-backed MCP server alive across many synchronous `chrome-devtools` CLI invocations. The CLI talks to it over a local socket (Unix domain socket / Windows named pipe); the daemon proxies each request to its own MCP server child over stdio. This avoids relaunching Chrome on every command.

## Key files
- `src/daemon/daemon.ts` — the daemon process body: PID file, socket server, MCP stdio child, request handling, cleanup.
- `src/daemon/client.ts` — CLI-side API: `startDaemon`, `stopDaemon`, `sendCommand`, `handleResponse`, `waitForFile`.
- `src/daemon/utils.ts` — path resolution (`getSocketPath`, `getPidFilePath`, `getRuntimeHome`), liveness (`isDaemonRunning`), `serializeArgs`.
- `src/daemon/types.ts` — `DaemonMessage` (`stop`|`status`|`invoke_tool`) and `DaemonResponse` IPC shapes.

## Key types & functions
- `DaemonMessage` — `{method:'stop'} | {method:'status'} | {method:'invoke_tool', tool, args?}`.
- `DaemonResponse` — `{success, result /* stringified CallToolResult */, error}`.
- `startDaemon(mcpArgs, sessionId)` / `stopDaemon(sessionId)` / `sendCommand(command, sessionId)`.
- `isDaemonRunning(sessionId)` — reads the PID file and `process.kill(pid, 0)`-probes liveness.
- `serializeArgs(options, argv)` — turns parsed yargs `argv` back into `--kebab-case` arg strings (booleans → `--flag`/`--no-flag`, arrays → repeated, others → `--k v`).

## How it works

### Path/identity scheme (`utils.ts`)
- `getSocketPath(sessionId)`: Windows → `\\.\pipe\<app><-session>\server.sock`; Unix → `$XDG_RUNTIME_DIR/<app>/server.sock` if set, else `/tmp/<app>-<uid>.sock` (chosen for the 104-char POSIX socket-path limit).
- `getRuntimeHome(sessionId)` → `$XDG_RUNTIME_DIR/<app>` or `/tmp/<app>-<uid>` (cleared on boot) or `os.tmpdir()/<app>` on Windows; `getPidFilePath` is `<runtimeHome>/daemon.pid`.
- `app = 'chrome-devtools-mcp'` plus `-<sessionId>` suffix when a session is given.
- `INDEX_SCRIPT_PATH` points at `../bin/chrome-devtools-mcp.js` (the bin shim) and `DAEMON_SCRIPT_PATH` at `daemon.js`.

### Daemon startup (`daemon.ts`)
The daemon process reads `CHROME_DEVTOOLS_MCP_SESSION_ID` from env, exits 1 if a daemon is already running, then **hardens the PID file**:
- creates the PID dir; on POSIX, `statSync` checks the dir is owned by the current uid and is **not** group/world-writable (tamper checks) — exits 1 otherwise.
- opens the PID file with `O_WRONLY|O_CREAT|O_TRUNC|O_NOFOLLOW` and mode `0o600` (refusing to follow symlinks), writes `process.pid`.

It then `startSocketServer()`: removes any stale socket (non-Windows), `createServer` with `readableAll:false, writableAll:false`, wraps each connection in a Puppeteer `PipeTransport`, and on `listen` calls `setupMCPClient()` — which spawns `process.execPath INDEX_SCRIPT_PATH ...mcpServerArgs` via `StdioClientTransport` and connects an MCP `Client` named `chrome-devtools-cli-daemon`. So the daemon is an MCP **client** of a `chrome-devtools-mcp` server child.

### Request handling
Each socket message is JSON-parsed into a `DaemonMessage` and dispatched in `handleRequest`:
- `invoke_tool` → `mcpClient.callTool({name, arguments})`, returns `{success:true, result: JSON.stringify(result)}`.
- `stop` → awaits the in-progress `started` promise, then `setImmediate(cleanup)`.
- `status` → returns `{pid, socketPath, startDate, version, args}`.
Errors are caught and returned as `{success:false, error}`. After responding, the socket is `.end()`ed (one request per connection).

### Client side (`client.ts`)
- `startDaemon`: clears a stale PID file, `spawn(execPath, [DAEMON_SCRIPT_PATH, ...mcpArgs], {detached, stdio:'ignore', env: {...process.env, CHROME_DEVTOOLS_MCP_SESSION_ID: sessionId}})`, `child.unref()`, then `waitForFile(pidFilePath)` (fs.watchFile poll, 10s timeout).
- `sendCommand`: opens a `net` connection, sends one `DaemonMessage` over `PipeTransport`, resolves on response (60s timeout, rejects on socket error/close).
- `stopDaemon`: `sendCommand({method:'stop'})` then `waitForFile(pid, removed=true)`.

### Cleanup
`cleanup()` (on SIGTERM/SIGINT/SIGHUP or a `stop` request) closes the MCP client + transport, closes the socket server, unlinks the socket (non-Windows) and the PID file, then `process.exit(0)`. `uncaughtException`/`unhandledRejection` are logged but not fatal.

## Relationships
- **Depends on:** the MCP server binary it spawns ([binaries.md](binaries.md) — `INDEX_SCRIPT_PATH`), `PipeTransport`/`Client`/`StdioClientTransport` (`src/third_party/index.ts`), `getTempFilePath` ([../08-utilities/](../08-utilities/)).
- **Used by:** the `chrome-devtools` CLI ([cli.md](cli.md)) — every tool subcommand and `start`/`stop`/`status`.

## Gotchas & non-obvious details
- The daemon runs the MCP server as a **stdio child**, then re-exposes it over a socket — two hops (CLI → socket → daemon → stdio → MCP server). The daemon itself is not an MCP server endpoint.
- POSIX socket paths must stay short (≤104 chars); that's why `/tmp/<app>-<uid>.sock` is used over `~/Library/...`.
- PID-file hardening (ownership/permission checks, `O_NOFOLLOW`) is a deliberate anti-tamper measure — do not relax it.
- `isDaemonRunning` treats a stale PID (process gone) as not running, so startup self-heals.
- One socket connection = one request (`socket.end()` after each response); the CLI opens a fresh connection per command.

## Update triggers
- The IPC protocol (`DaemonMessage`/`DaemonResponse`) or `invoke_tool`/`status`/`stop` handling changes.
- Socket/PID path resolution or the XDG/`/tmp`/Windows-pipe scheme changes.
- The daemon's MCP child spawn (`INDEX_SCRIPT_PATH`, client name, env) or PID-file hardening changes.
