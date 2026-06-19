# Telemetry & Logging

> **Layer:** Telemetry & logging · **Sources:** `src/telemetry/`, `src/logger.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Collects anonymous usage statistics (server start, tool invocations, daily-active pings, server errors) and ships them to Google's Clearcut logging endpoint via a **separate detached watchdog process**, so telemetry survives and flushes even when the main MCP server dies. Also provides the `debug`-based namespaced logger used everywhere.

## Key files
- `src/telemetry/ClearcutLogger.ts` — singleton API the server calls to log events
- `src/telemetry/types.ts` — event/IPC/Clearcut wire types and enums (`McpClient`, `OsType`, `WatchdogMessageType`)
- `src/telemetry/WatchdogClient.ts` — spawns the detached watchdog and writes IPC lines to its stdin
- `src/telemetry/watchdog/main.ts` — watchdog process entrypoint (reads stdin, watches for parent death)
- `src/telemetry/watchdog/ClearcutSender.ts` — buffers events, batches, retries, POSTs to Clearcut
- `src/telemetry/transformation.ts` — param sanitization, bucketization, name/type transforms
- `src/telemetry/metricsRegistry.ts` — build-time tool-metric schema generation
- `src/telemetry/flagUtils.ts` — CLI flag-usage metrics
- `src/telemetry/persistence.ts` — on-disk `telemetry_state.json` (daily-active tracking)
- `src/telemetry/errors.ts` — `ErrorCode` enum
- `src/logger.ts` — `debug('mcp:log')` logger + log-to-file helpers

## Documentation index
- [clearcut-logger.md](clearcut-logger.md) — `ClearcutLogger`: events, when each is logged
- [watchdog.md](watchdog.md) — separate sender process, why it exists, IPC, batching/retry
- [metrics-registry.md](metrics-registry.md) — `metricsRegistry`, `transformation`, `ErrorCode`
- [flags-and-persistence.md](flags-and-persistence.md) — flag-usage metrics + on-disk state
- [logging.md](logging.md) — the `debug` logger and `--log-file`

## How it works
1. `createMcpServer` ([server bootstrap](../01-entrypoints/server-bootstrap.md)) calls `ClearcutLogger.initialize()` **only if** `serverArgs.usageStatistics` is true (`src/index.ts:38`).
2. `ClearcutLogger` (singleton) constructs a `WatchdogClient`, which `spawn`s `watchdog/main.js` as a **detached, unref'd** child with `stdio: ['pipe','ignore','ignore']`.
3. Server code calls `logServerStart` / `logToolInvocation` / `logDailyActiveIfNeeded` / `logServerError`. Each serializes a `WatchdogMessage` and writes it as a newline-delimited JSON line to the watchdog's stdin.
4. The watchdog's `ClearcutSender` enriches each event with `session_id`/`app_version`/`os_type`, buffers it (max 1000), and flushes batches to `https://play.googleapis.com/log?format=json_proto` on a timer (default 15 min), with rate-limit and transient-error retry.
5. When the parent dies (stdin end/close or IPC disconnect), the watchdog sends a final `server_shutdown` event and exits.

## Relationships
- **Depends on:** [CLI options](../01-entrypoints/cli.md) (the `--no-usage-statistics` flag + env opt-out), `src/third_party` (the `debug` lib + zod re-exports), [tools](../02-tools/README.md) (`ToolDefinition` schemas for param sanitization).
- **Used by:** [server bootstrap](../01-entrypoints/server-bootstrap.md) (`src/index.ts` initializes + sets client name), [ToolHandler](../02-tools/tool-handler.md) (`src/ToolHandler.ts:296` logs every invocation in `finally`), [CLI main](../01-entrypoints/binaries.md) (`src/bin/chrome-devtools-mcp-main.ts` logs server start + daily active), `persistence.ts` (logs its own file-IO errors).

## Gotchas & non-obvious details
- **Opt-out is enforced at parse time, not in the telemetry layer.** `ClearcutLogger.initialize()` is simply never called when disabled. Disabled by `--no-usage-statistics`, or env `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS`, or `CI` (`src/bin/chrome-devtools-mcp-cli-options.ts:380`).
- **No PII / no raw values.** String params are reduced to bucketized lengths, arrays to counts; `uid`/`reqid`/`msgid` are blocklisted. See [metrics-registry.md](metrics-registry.md).
- **`session_id` is an ephemeral random UUID** generated in the watchdog, rotated every 24h — not a stable user identifier.
- **Fire-and-forget:** all logging calls are `void`-ed / wrapped so telemetry failures never crash the server. Watchdog stdin write failures just log + drop.

## Update triggers
- A new event type is added to `ChromeDevToolsMcpExtension` / a new `logX` method on `ClearcutLogger`.
- `metricsRegistry`, `flagUtils`, or `transformation` change (param sanitization, buckets, blocklist).
- The Clearcut endpoint, `LOG_SOURCE`/`CLIENT_TYPE`, or watchdog IPC/lifecycle changes.
- The opt-out flag/env vars change.
