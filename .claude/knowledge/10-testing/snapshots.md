# Snapshot Testing

> **Layer:** Testing · **Sources:** `tests/setup.ts`, `tests/utils.ts`, `tests/McpContext.test.js.snapshot`, `scripts/test.mjs`, `package.json` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
How the suite uses `node:test`'s built-in snapshot assertion to lock down large
formatted outputs (tool responses, formatters, trace parsing, generated tool
lists) and how to regenerate them.

## Key files
- `tests/setup.ts` — configures snapshot path + serializer
- `tests/utils.ts` — `stabilizeResponseOutput` / `stabilizeStructuredContent` normalizers
- `tests/**/*.test.js.snapshot` — committed snapshot files (e.g. `tests/tools/console.test.js.snapshot`)

## How it works
Snapshots use `node:test`'s native API: inside a test, `t.assert.snapshot(value)`
compares `value` against a stored expectation, writing the file on first run or
when `--test-update-snapshots` is set.

```ts
it('lists error objects', async t => {
  // …produce textContent…
  t.assert.snapshot(textContent);
});
```

### Where snapshots live (and why)
By default node writes snapshots next to the **compiled** file in `build/tests/`.
`tests/setup.ts` overrides this so they land in source `tests/`:

```ts
it.snapshot.setResolveSnapshotPath(testPath =>
  testPath?.replace(path.join('build','tests'), 'tests') + '.snapshot');
it.snapshot.setDefaultSnapshotSerializers([String]);
```

The `String` serializer is chosen over the default `JSON.stringify` so multi-line
text renders with real newlines — making `.snapshot` files human-readable diffs.
Each entry is keyed `exports[\`<describe> > <it> N\`]` and stored as a template
literal.

### Stabilizing nondeterministic output
Raw output contains volatile bits (dates, random ports, user-agent, file paths).
Tests normalize before snapshotting:
- `stabilizeResponseOutput(text)` — regex-replaces dates → `<long date>`,
  `localhost:NNNNN` → `localhost:<port>`, UA/`sec-ch-ua`/`accept-language`
  headers, `Saved snapshot to …` paths, and URL-encoded `file://` paths.
- `stabilizeStructuredContent(obj)` — recurses JSON and replaces
  `snapshotFilePath` values with `<file>`.
- Ad-hoc `replaceAll` in individual tests (e.g. masking a live `extensionId`,
  `reqid=\d+`, `ID: \d+`).

### Commands
```bash
npm run test:update-snapshots          # rebuild + regenerate all snapshots
npm run test:update-snapshots tests/tools/console.test.ts  # one file
```
`scripts/test.mjs` simply forwards `--test-update-snapshots` to node's runner.

## Relationships
- **Tests:** formatter output (`src/formatters/*`), tool responses (`src/tools/*`), `src/trace-processing/`, generated tool metadata (`tests/tools/slim/`, `index.test.ts`).
- **Depends on:** [Build tooling](../09-build-tooling/README.md); snapshot path remap in `tests/setup.ts`.

## Gotchas & non-obvious details
- Always build first — updating snapshots from stale `build/` writes wrong expectations.
- If a diff is only volatile data (port/date/UA), the fix is usually a missing stabilizer call, not a real change.
- Large committed snapshots exist (`tests/third_party_notices.test.js.snapshot` ~180 KB) — expect big diffs when deps change.
- Review snapshot diffs deliberately; blind `:update-snapshots` can mask real regressions.

## Update triggers
- A formatter or tool output format intentionally changes.
- New volatile field appears in output (add a stabilizer).
- The snapshot serializer or path-resolution logic in `tests/setup.ts` changes.
