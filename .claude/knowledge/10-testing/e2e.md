# End-to-End Tests

> **Layer:** Testing · **Sources:** `tests/e2e/`, `tests/utils.ts` (`runCli`, daemon asserts) · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`tests/e2e/` exercises the shipped binaries as **black-box subprocesses** — the
`chrome-devtools` CLI, the daemon lifecycle, and the `chrome-devtools-mcp`
server's telemetry pipeline — rather than importing functions in-process.

## Key files
- `chrome-devtools-start-stop.test.ts` — daemon `start` / `stop` (and `--userDataDir`)
- `chrome-devtools-status.test.ts` — `status` reporting
- `chrome-devtools-commands.test.ts` — invoking tools through the CLI (e.g. `list_pages`)
- `chrome-devtools-disclaimers.test.ts` — disclaimer/notice output
- `telemetry.test.ts` — spins a mock Clearcut HTTP endpoint, runs the server, asserts emitted events

## How it works
These tests spawn the **built** binaries and assert on exit code + stdout/stderr,
so they validate real process wiring (arg parsing, daemon sockets, IPC,
telemetry HTTP) that unit tests stub out.

CLI/daemon tests use the `runCli(args, sessionId)` helper from `tests/utils.ts`,
which spawns `build/src/bin/chrome-devtools.js`. Each test generates a fresh
`crypto.randomUUID()` **sessionId** and brackets the body with `stop` in
`beforeEach`/`afterEach` so daemons never leak between tests:

```ts
beforeEach(async () => {
  sessionId = crypto.randomUUID();
  await runCli(['stop'], sessionId);
  await assertDaemonIsNotRunning(sessionId);
});
```

`assertDaemonIsRunning` / `assertDaemonIsNotRunning` assert on `status` stdout.

`telemetry.test.ts` is heavier: it stands up a local `node:http` server posing as
the Clearcut collector, launches `build/src/bin/chrome-devtools-mcp.js`, parses
the `source_extension_json` log events it receives, and uses a `waitForEvent`
predicate to await specific telemetry. It also tracks the watchdog PID via the
`x-watchdog-pid` header.

## How they differ from unit tests
- **Process boundary:** real subprocess + real exit codes, not direct `handler()` calls.
- **No `withMcpContext`:** they don't build an in-memory `McpContext`; they talk to a running daemon/server.
- **Slower & statefuller:** start/stop real daemons and Chrome; isolation comes from per-test sessionIds, not browser pooling.
- **Run last / retried:** flakiness here is handled by `scripts/test.mjs --retry` (up to 3 attempts) in CI.

## Relationships
- **Tests:** `src/bin/` (CLI + server entrypoints), `src/daemon/`, `src/telemetry/` end-to-end. See [Entrypoints](../01-entrypoints/README.md) and [Telemetry](../07-telemetry/README.md).
- **Depends on:** built binaries, a Chrome binary, [Build tooling](../09-build-tooling/README.md). Complements unit-level daemon tests in `tests/daemon/`.

## Gotchas & non-obvious details
- Always `npm run test` (build) first — e2e runs the compiled binaries, so stale `build/` tests stale behavior.
- Leftover daemons from a crashed run can wedge later tests; the per-sessionId `stop` brackets guard against this but a manual `chrome-devtools stop` may be needed.
- The runner sets `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS`, `..._CRASH_ON_UNCAUGHT`, `..._NO_UPDATE_CHECKS` for all tests — telemetry e2e overrides the endpoint to its mock server rather than relying on the disable flag.

## Update triggers
- New CLI command, daemon behavior, or telemetry event shape.
- Changes to binary paths in `package.json` `bin` or to `runCli`.
