# Watchdog (separate sender process)

> **Layer:** Telemetry & logging · **Sources:** `src/telemetry/WatchdogClient.ts`, `src/telemetry/watchdog/main.ts`, `src/telemetry/watchdog/ClearcutSender.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The watchdog is a **separate, detached Node child process** that owns all telemetry buffering and network I/O. It exists so events still flush even after the main MCP server exits, and so blocking/slow HTTP never affects the server. The main process only writes newline-delimited JSON to the watchdog's stdin.

## Key files
- `src/telemetry/WatchdogClient.ts` — runs in the **main** process; spawns + feeds the watchdog
- `src/telemetry/watchdog/main.ts` — the **child** process entrypoint
- `src/telemetry/watchdog/ClearcutSender.ts` — buffering/batching/retry + the actual `fetch` to Clearcut

## Key types & functions
- `WatchdogClient` — constructor `spawn`s the child; `send(message)` writes one JSON line to its stdin.
- `ClearcutSender` — `enqueueEvent(event)`, `sendShutdownEvent()`, internal `#flush` / `#sendBatch` / `#finalFlush`.

## How it works

### Spawning (`WatchdogClient`)
Resolves `./watchdog/main.js` relative to itself, then:
```ts
this.#childProcess = spawner(process.execPath, args, {
  stdio: ['pipe', 'ignore', 'ignore'],
  detached: true,
});
this.#childProcess.unref();
```
`detached: true` + `unref()` mean the child does **not** keep the parent's event loop alive and can outlive it for the final flush. CLI args carry `parent-pid`, `app-version`, `os-type`, and optional `log-file` / `clearcut-endpoint` / `clearcut-force-flush-interval-ms` / `clearcut-include-pid-header`. `send()` writes `JSON.stringify(message) + '\n'`; if stdin is gone it logs and drops the message (never throws).

### IPC (newline-delimited JSON over stdin)
`main.ts` uses `readline` over `process.stdin`; each line is `JSON.parse`d and, if `type === LOG_EVENT`, the payload is passed to `sender.enqueueEvent`. Parse errors are logged and skipped. There is no reverse channel — the watchdog never writes back to the parent.

### Parent-death detection
`onParentDeath` fires once (guarded by `isShuttingDown`) on any of: stdin `end`, stdin `close`, or process `disconnect`. It calls `sender.sendShutdownEvent()` then exits. This is how the `server_shutdown` event is generated.

### Buffering, batching, retry (`ClearcutSender`)
- On `enqueueEvent`: rotates `#sessionId` if older than 24h, enriches the event with `session_id` / `app_version` / `os_type`, pushes to `#buffer` (cap `MAX_BUFFER_SIZE = 1000`; oldest dropped on overflow), and starts the flush timer on first event.
- Flush timer default `DEFAULT_FLUSH_INTERVAL_MS = 15 min` (overridable via `--clearcut-force-flush-interval-ms`, used by tests).
- `#flush` optimistically drains the buffer before sending (avoids double-send races with `#finalFlush`). `#isFlushing` guards re-entrancy.
- `#sendBatch` POSTs a `LogRequest` (`log_source: 2839`, `client_info.client_type: 47`) to `DEFAULT_CLEARCUT_ENDPOINT = https://play.googleapis.com/log?format=json_proto`, 30s `AbortController` timeout.
- Response handling: `ok` → success, honor `next_request_wait_millis` (floored at `MIN_RATE_LIMIT_WAIT_MS = 30s`). Status `>= 500` or `429` → transient, requeue at front of buffer. Other 4xx → permanent error, **drop the batch**. Thrown errors → requeue.
- `sendShutdownEvent` clears the timer, enqueues `{server_shutdown: {}}`, then races `#finalFlush()` against `SHUTDOWN_TIMEOUT_MS = 5s`.

## Relationships
- **Depends on:** `types.ts` (`WatchdogMessage`, `ChromeDevToolsMcpExtension`, `OsType`), `logger.ts` (+ `saveLogsToFile`/`flushLogs` for `--log-file`). Network: Google Clearcut.
- **Used by:** [ClearcutLogger](clearcut-logger.md) (constructs `WatchdogClient`, the only writer).

## Gotchas & non-obvious details
- The watchdog is its own process: a built `main.js` must exist next to the compiled `WatchdogClient.js`. Failure to spawn logs an `error` event and silently disables telemetry.
- `--clearcut-include-pid-header` adds an `X-Watchdog-Pid` request header — **used by E2E tests** to confirm the watchdog process was actually killed, not a production feature.
- On permanent (non-429/non-5xx) errors the batch is **discarded**, so persistent 4xx config errors lose events rather than retrying forever.
- `session_id` is a random UUID rotated every 24h — ephemeral, not a stable user/device ID.
- `stopForTesting()` / `bufferSizeForTesting` exist solely for unit tests.

## Update triggers
- The Clearcut endpoint, `LOG_SOURCE`, `CLIENT_TYPE`, or request shape changes.
- Buffer cap, flush interval, timeouts, or retry classification change.
- Parent-death detection or the IPC line format changes.
