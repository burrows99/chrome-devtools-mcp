# Fixtures & Test Server

> **Layer:** Testing · **Sources:** `tests/server.ts`, `tests/snapshot.ts`, `tests/fixtures/`, `tests/tools/fixtures/`, `tests/trace-processing/fixtures/load.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
How tests serve content and supply on-disk inputs: a per-suite local HTTP server
plus three flavors of static fixture (extensions, traces, heap snapshots) and
inline HTML helpers.

## Key files
- `tests/server.ts` — `TestServer` class + `serverHooks()` lifecycle helper
- `tests/utils.ts` — `html\`…\`` tagged template (wraps a body in a full HTML doc)
- `tests/snapshot.ts` — `screenshots` record of tiny inline HTML pages for screenshot tests
- `tests/tools/fixtures/extension*/` — unpacked MV3 extensions
- `tests/fixtures/example.heapsnapshot` — a real `.heapsnapshot` for memory tests
- `tests/trace-processing/fixtures/` — gzipped traces + `load.ts` decompressor

## How it works

### TestServer
`TestServer` is a thin `node:http` server. `TestServer.randomPort()` picks a
port in 10101–20202 (avoiding Chromium's restricted ports). Routes are
registered on demand: `addHtmlRoute(path, html)` and `addRoute(path, handler)`;
unknown paths return 404. `getRoute('/x')` returns the absolute URL.

`serverHooks()` is the idiomatic entry: it constructs a server on a random port
and wires `before(start)` / `after(stop)` / `afterEach(restore)` so each test
sees a clean route table. Usage:

```ts
const server = serverHooks();
server.addHtmlRoute('/index.html', `<script src="${server.getRoute('/main.min.js')}"></script>`);
await page.pptrPage.goto(server.getRoute('/index.html'));
```

For tests that don't need a route registry, `page.pptrPage.setContent(html\`…\`)`
loads markup directly (the `html` tag in `tests/utils.ts` wraps the body in a
full `<!DOCTYPE html>` document).

### Extension fixtures
`tests/tools/fixtures/extension*/` are unpacked Manifest V3 extensions
(`manifest.json` + `sw.js`/`popup.html`/`content.js`). Variants cover plain,
service-worker, content-script, side-panel, and logging cases. Tests pass the
fixture directory to `installExtension.handler(...)` and read back the installed
extension id with `extractExtensionId(response)`.

### Trace & heap fixtures
`tests/trace-processing/fixtures/*.json.gz` are gzipped Chrome traces;
`load.ts`'s `loadTraceAsBuffer()` gunzips and returns a `Uint8Array` for the
parser. `tests/fixtures/example.heapsnapshot` is a ~2.3 MB real snapshot for
`HeapSnapshotFormatter`/memory tests.

### Fixture path resolution (important)
Fixtures are **not** copied into `build/`. Compiled tests live at e.g.
`build/tests/tools/console.test.js`, so they walk back up to the source tree:

```ts
path.join(import.meta.dirname, '../../../tests/tools/fixtures/extension-logging')
```

`load.ts` walks four `..` segments for the same reason. If you move a fixture
or change `outDir`, these relative hops break.

## Relationships
- **Tests:** `src/tools/extensions.ts`, network/console tools, `src/trace-processing/`, `src/HeapSnapshotManager.ts`.
- **Depends on:** [Build tooling](../09-build-tooling/README.md) (compiled test location dictates the `../../../` hops).

## Gotchas & non-obvious details
- `addRoute`/`addHtmlRoute` throw if a path is registered twice; rely on `afterEach` restore between tests.
- Random ports can theoretically collide; the range deliberately dodges Chromium-restricted ports.
- Fixtures sit outside `build/`; the brittle `../../../` paths are the price of not copying them.

## Update triggers
- New fixture type, or a fixture directory move/rename.
- `outDir`/`rootDir` change in tsconfig (rebalances the `..` path hops).
- Changes to `TestServer` API or `serverHooks` lifecycle.
