# Logging (`src/logger.ts`)

> **Layer:** Telemetry & logging · **Sources:** `src/logger.ts`, `src/types.ts` (`Logger` type) · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
A thin wrapper over the `debug` library giving the whole codebase one namespaced logger (`mcp:log`), plus helpers to redirect logs to a file. This is the **diagnostic/debug** logger — entirely separate from the Clearcut telemetry pipeline (it stays local, nothing is sent anywhere).

## Key files
- `src/logger.ts` — the `logger` export + `saveLogsToFile` / `flushLogs`
- `src/third_party/index.ts` — re-exports the `debug` lib used here
- `src/types.ts` — `Logger` type alias

## Key types & functions
- `logger` — `debug('mcp:log')`, typed as `Logger`. The app-wide logging function; imported nearly everywhere as `logger?.(...)`.
- `Logger` (`src/types.ts:37`) — `((...args: unknown[]) => void) | undefined`. The optional-call typing is why call sites use `logger?.(...)`.
- `saveLogsToFile(fileName)` → `fs.WriteStream` — enable namespaces, append all `debug` output to a file.
- `flushLogs(logFile, timeoutMs = 2000)` → `Promise<void>` — flush + close the stream (with a reject-on-timeout).

## How it works
The namespace `mcp:log` is always enabled; if `process.env.DEBUG` is set it's appended, so users can widen logging with the standard `DEBUG` env var.

`saveLogsToFile` calls `debug.enable(namespacesToEnable.join(','))` (because enabling overrides any prior config), opens the file in append mode (`flags: 'a+'`), and **monkey-patches `debug.log`** to write `chunks.join(' ') + '\n'` to that stream:
```ts
debug.log = function (...chunks: any[]) {
  logFile.write(`${chunks.join(' ')}\n`);
};
```
A stream `error` handler logs to `console.error`, ends the stream, and `process.exit(1)` — a file-logging error is treated as fatal. `flushLogs` resolves once `logFile.end()`'s callback fires.

This file logging is opt-in via the `--log-file` CLI flag, plumbed through to both the main process and the [watchdog](watchdog.md) (`WatchdogClient` passes `--log-file`, and `watchdog/main.ts` calls `saveLogsToFile`/`flushLogs` so watchdog diagnostics land in the same file).

## Relationships
- **Depends on:** the `debug` package (via `src/third_party`), `node:fs`.
- **Used by:** essentially the entire codebase via `logger?.(...)`; within this layer: `ClearcutLogger`, `WatchdogClient`, `ClearcutSender`, `persistence`, `watchdog/main`. `saveLogsToFile`/`flushLogs` are called from [CLI/server bootstrap](../01-entrypoints/server-bootstrap.md) and `watchdog/main.ts`.

## Gotchas & non-obvious details
- **Not telemetry.** This logger writes locally (stderr by default, or `--log-file`); it does not feed Clearcut. Reviewers conflating "logging" with "usage statistics" should note the separation.
- `logger` is typed as possibly `undefined`, hence the universal `logger?.(...)` idiom — but in practice `debug(...)` always returns a function.
- `debug.log` is monkey-patched **globally**, so once a log file is configured all `debug` output (any namespace) goes to the file, not just `mcp:log`.
- A failure writing the log file is **fatal** (`process.exit(1)`).

## Update triggers
- The log namespace, `DEBUG` handling, or file-logging behavior changes.
- The `Logger` type or the `logger?.(...)` calling convention changes.
