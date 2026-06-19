# ClearcutLogger

> **Layer:** Telemetry & logging · **Sources:** `src/telemetry/ClearcutLogger.ts`, `src/telemetry/types.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`ClearcutLogger` is the singleton API the MCP server calls to record telemetry events. It does **no buffering or network I/O itself** — it builds an event object and hands it to a `WatchdogClient` over IPC. The actual batching/sending happens in the [watchdog](watchdog.md).

## Key files
- `src/telemetry/ClearcutLogger.ts` — the singleton class
- `src/telemetry/types.ts` — event payload types (`ChromeDevToolsMcpExtension`) and enums

## Key types & functions
- `ClearcutLogger.initialize(options)` — creates the singleton; **throws if already initialized**. Constructs a `WatchdogClient` unless one is injected (testing).
- `ClearcutLogger.get()` — returns the singleton or `undefined` (undefined when usage statistics are off, since `initialize` was never called).
- `ClearcutLogger.resetForTesting()` — clears the singleton.
- `setClientName(clientName)` — maps the connecting MCP client name to a `McpClient` enum, stored as `#mcpClient` and attached to every event.
- `logServerStart(flagUsage)`, `logToolInvocation(args)`, `logDailyActiveIfNeeded()`, `logServerError(args)` — the four event-logging methods.

## How it works

### Singleton lifecycle
Module-level `_clearcut_logger_instance`; constructor is `private`. Initialized once in `createMcpServer` (`src/index.ts:39`) and only when `serverArgs.usageStatistics` is true. Three private fields: `#persistence`, `#watchdog`, `#mcpClient` (starts `MCP_CLIENT_UNSPECIFIED`).

### Client detection (`setClientName`)
Lower-cases the client name and substring-matches to `McpClient`:
- `claude` → `MCP_CLIENT_CLAUDE_CODE`, `gemini` → `MCP_CLIENT_GEMINI_CLI`, `openclaw` → `OPENCLAW`, `codex` → `CODEX`, `antigravity` → `ANTIGRAVITY`.
- Exact match on `DAEMON_CLIENT_NAME` → `MCP_CLIENT_DT_MCP_CLI`.
- Anything else → `MCP_CLIENT_OTHER`. Called from `server.server.oninitialized` (`src/index.ts:79`).

### Events logged
Every event is sent as `{type: WatchdogMessageType.LOG_EVENT, payload: ChromeDevToolsMcpExtension}` with `mcp_client` set, via `this.#watchdog.send(...)`.

- **`server_start`** (`logServerStart`) — carries `flag_usage` (see [flags-and-persistence.md](flags-and-persistence.md)). Called once from CLI main.
- **`tool_invocation`** (`logToolInvocation`) — `{tool_name, success, latency_ms, tool_params?}`. `tool_name` is run through `stripUnderscoreBeforeNumber`. Params are added only if non-empty, nested under a `${toolName}_params` key, and sanitized via `sanitizeParams` (lengths/counts, blocklist — see [metrics-registry.md](metrics-registry.md)). Latency is already bucketized by the caller (`bucketizeLatency`).
- **`daily_active`** (`logDailyActiveIfNeeded`) — `{days_since_last_active}`. Reads `telemetry_state.json` via `#persistence`; logs only if `lastActive` is empty or not the same UTC calendar day. Computes `daysSince` (= `-1` for first-ever), then writes the new `lastActive` ISO timestamp. Wrapped in try/catch — never throws.
- **`server_error`** (`logServerError`) — `{tool_name, error_code}`. `tool_name` defaults to `''`; `error_code` is an `ErrorCode`. Currently emitted only by `FilePersistence` on read/save failures.
- **`server_shutdown`** — note: **not** sent by `ClearcutLogger`. The watchdog synthesizes it on parent death (see [watchdog.md](watchdog.md)).

## Relationships
- **Depends on:** `WatchdogClient` ([watchdog.md](watchdog.md)), `Persistence` ([flags-and-persistence.md](flags-and-persistence.md)), `transformation.ts` (`sanitizeParams`, `stripUnderscoreBeforeNumber`), `errors.ts` (`ErrorCode`), `logger.ts`.
- **Used by:** [server bootstrap](../01-entrypoints/server-bootstrap.md) (`src/index.ts`), [ToolHandler](../02-tools/tool-handler.md) (`src/ToolHandler.ts:296`, in a `finally`), [CLI main](../01-entrypoints/binaries.md).

## Gotchas & non-obvious details
- The `logX` methods are `async` but resolve immediately — `#watchdog.send` is synchronous (a stdin write). The `async` is vestigial; callers `void` them anyway.
- `logToolInvocation` is called in `ToolHandler`'s `finally`, so it fires on both success and failure (`success` reflects which).
- `detectOsType()` maps `process.platform` to `OsType`; unknown platforms → `OS_TYPE_UNSPECIFIED`.
- The numeric enum values are **non-contiguous** (e.g. `MCP_CLIENT_OTHER = 3`, `DT_MCP_CLI = 4`) and must match the server-side protobuf — do not renumber.

## Update triggers
- A new `logX` method or a new field on `ChromeDevToolsMcpExtension`.
- A new MCP client added to `setClientName` / the `McpClient` enum.
- The daily-active "same UTC day" logic changes.
