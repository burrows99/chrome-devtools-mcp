# Test Architecture

> **Layer:** Testing · **Sources:** `tests/setup.ts`, `tests/utils.ts`, `tests/McpContext.test.ts`, `tests/tools/console.test.ts`, `tests/tools/slim/tools.test.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The framework, harness, and recurring patterns used across the suite. Confirmed:
tests use Node's **built-in `node:test`** (`describe`/`it`/`before`/`after`/
`afterEach`), `node:assert`, and **sinon** for spies/stubs. No third-party test
framework is present.

## Key files
- `tests/setup.ts` — global setup loaded once via `node --import`
- `tests/utils.ts` — the harness: `withBrowser`, `withMcpContext`, mocks, stabilizers, CLI helpers

## How it works

### Global setup (`tests/setup.ts`)
Loaded with `--import ./build/tests/setup.js` so it runs before any test. It:
1. imports `src/polyfill.js`,
2. registers a `before()` that stubs DevTools globals (`overrideDevToolsGlobals`),
3. rewrites the snapshot path from `build/tests/` back to `tests/` and sets a
   `String` snapshot serializer (see [snapshots.md](./snapshots.md)).

### Driving a real browser: `withMcpContext`
The dominant pattern. Most tool/context tests call:

```ts
await withMcpContext(async (response, context) => {
  const page = context.getSelectedMcpPage();
  await page.pptrPage.setContent(html`<button>Click me</button>`);
  await listConsoleMessages().handler(
    {params: {}, page: context.getSelectedMcpPage()},
    response,
    context,
  );
  const formatted = await response.handle('test', context);
  assert.ok(getTextContent(formatted.content[0]).includes('…'));
});
```

`withMcpContext` (in `tests/utils.ts`) wraps `withBrowser`, resets
`TextSnapshot` counters, builds a fresh `McpResponse` + `McpContext.from(...)`,
selects a page, and invokes the callback. Tools are exercised by importing the
real tool definition from `src/tools/*` and calling `.handler({params, page},
response, context)`, then asserting on `response.handle(...)` output via
`getTextContent` / `getImageContent` or `structuredContent`.

### Real Chrome, pooled
`withBrowser` launches Puppeteer **headless** (`headless: !options.debug`,
`pipe: true`, `enableExtensions: true`) and **caches browsers by launch-option
JSON key** in a module-level `Map`, reusing them across tests for speed. Each
invocation opens a fresh page and closes all others. `executablePath` falls back
to `PUPPETEER_EXECUTABLE_PATH`. Options expose `blockedUrlPattern`,
`allowedUrlPattern`, `autoOpenDevTools`, `args`.

### Pure-unit mocking with sinon
Tests that don't need a browser use sinon stubs and the hand-rolled mocks in
`tests/utils.ts`: `getMockRequest` / `getMockResponse` (fake Puppeteer
`HTTPRequest`/`HTTPResponse`), `getMockPage` / `getMockBrowser`,
`getMockAggregatedIssue`, and `mockListener` (a tiny event-emitter). `sinon.stub`
/ `sinon.spy` patch context methods; `afterEach(() => sinon.restore())` is the
standard cleanup.

### CLI subprocess testing
`runCli(args, sessionId)` spawns the built `chrome-devtools` binary
(`build/src/bin/chrome-devtools.js`) and captures status/stdout/stderr.
`assertDaemonIsRunning` / `assertDaemonIsNotRunning` wrap `runCli(['status'])`.
Used by daemon and e2e tests.

## Relationships
- **Tests:** `src/McpContext.ts`, `src/McpResponse.ts`, `src/tools/*`, plus all root `src` classes.
- **Depends on:** Puppeteer/Chrome, sinon, [Build tooling](../09-build-tooling/README.md).

## Gotchas & non-obvious details
- Browsers are pooled by options key — a test that mutates global browser state can leak into the next.
- Many "tool" tests are really integration tests against live Chrome, so they are slower and need a Chrome binary.
- `tests/tools/slim/` tests the reduced `src/tools/slim/` toolset via the same `withMcpContext` harness.

## Update triggers
- New shared helper or mock added to `tests/utils.ts`.
- A switch away from `node:test`/sinon, or changes to `withMcpContext` signature.
