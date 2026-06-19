# 10 · Testing

> **Layer:** Testing · **Sources:** `scripts/test.mjs`, `package.json`, `tests/` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
How the test suite is built, run, and organized. Tests use the **built-in
`node:test` runner** (no Jest/Mocha/Vitest) driven by a thin wrapper at
`scripts/test.mjs`. Most tests launch a **real headless Chrome** via Puppeteer.

## Key files
- `scripts/test.mjs` — runner wrapper: maps `.ts` → `build/.../.js`, sets node flags, supports `--retry`
- `tests/setup.ts` — preloaded via `--import`; polyfills, DevTools globals, snapshot path/serializer config
- `tests/server.ts` — `TestServer`, an on-demand local HTTP server for fixtures
- `tests/utils.ts` — shared helpers (`withMcpContext`, `withBrowser`, mocks, `runCli`, stabilizers)
- `tests/snapshot.ts` — small inline HTML fixtures for screenshot tests

## How it works
Tests are written in TypeScript under `tests/`, compiled by `tsc` into
`build/tests/`, then run from the compiled `.js`. **You must build before
running** — `npm run test` does both:

```bash
npm run test                          # tsc build + run ALL tests
npm run test tests/tools/console.test.ts   # build + run ONE file (.ts path is remapped to build/)
npm run test:no-build tests/tools/console.test.ts  # skip build (faster when build is current)
npm run test:only                     # build + run only tests marked .only(...)
npm run test:update-snapshots         # build + regenerate *.test.js.snapshot files
```

`scripts/test.mjs` rewrites each `.ts` argument to its `build/.../.js` twin,
then spawns `node --import ./build/tests/setup.js --test --test-force-exit
--test-timeout=120000 …`. With no file args it globs `build/tests/**/*.test.js`.
Add `--retry` to re-run up to 3 times on failure (used in CI for flaky e2e).
Reporter is `dot` locally, `spec` under `CI`/`NODE_TEST_REPORTER`.

Directory layout under `tests/` mirrors `src/`: `tests/tools/` ↔ `src/tools/`,
`tests/formatters/` ↔ `src/formatters/`, `tests/telemetry/` ↔ `src/telemetry/`,
`tests/daemon/` ↔ `src/daemon/`, `tests/trace-processing/` ↔
`src/trace-processing/`, plus root-level files for the root `src/*.ts` classes
(`McpContext`, `McpResponse`, `TextSnapshot`, `PageCollector`, …).

## Index of this layer
- [test-architecture.md](./test-architecture.md) — node:test usage, setup, sinon, the `withMcpContext` pattern
- [fixtures-and-test-server.md](./fixtures-and-test-server.md) — `TestServer`, extension/HTML/trace/heapsnapshot fixtures
- [snapshots.md](./snapshots.md) — the `.test.js.snapshot` mechanism + when/how to update
- [e2e.md](./e2e.md) — `tests/e2e/` CLI/daemon/telemetry end-to-end tests

## Relationships
- **Tests:** every `src/` layer; see per-dir mapping above.
- **Depends on:** [Build tooling](../09-build-tooling/README.md) (must `tsc` first), Puppeteer + a Chrome binary.

## Gotchas & non-obvious details
- Build-before-test is mandatory; running stale `build/` silently tests old code.
- Fixtures are **not** copied into `build/`; tests walk back up to source `tests/` (see fixtures doc).
- Single-file runs still build the whole project unless you use `test:no-build`.

## Update triggers
- A new `src/` module/dir that needs a matching `tests/` dir.
- Changes to `scripts/test.mjs` flags, node version support, or the build step.
