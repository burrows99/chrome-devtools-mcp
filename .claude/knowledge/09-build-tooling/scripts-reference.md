# Scripts Reference

> **Layer:** Build & tooling · **Sources:** `package.json` (scripts), `scripts/*` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Single index of every file under `scripts/` and the npm-script that drives it. Detailed behavior lives in the linked focused docs.

## Key files / npm scripts

| npm script | Runs | Purpose |
|---|---|---|
| `build` | `tsc && scripts/post-build.ts` | Compile to `build/` + write DevTools shims → [build-and-bundle](build-and-bundle.md) |
| `clean` | inline `rm -rf build` | Delete `build/` |
| `bundle` | clean → build → `rollup` → rm `build/node_modules` → `append-lighthouse-notices.ts` | Publishable dependency-free bundle |
| `typecheck` | `tsc --noEmit` | Type-only check |
| `gen` | build → `docs:generate` → `cli:generate` → `update-metrics` → `format` | Full regeneration → [README](README.md) |
| `docs:generate` | `scripts/generate-docs.ts` | Tool reference + README regions → [code-generation](code-generation.md) |
| `cli:generate` | `scripts/generate-cli.ts` | `src/bin/chrome-devtools-cli-options.ts` → [code-generation](code-generation.md) |
| `update-metrics` | `scripts/update_metrics.ts` | Telemetry metrics JSON → [eval-and-metrics](eval-and-metrics.md) |
| `format` / `check-format` | `eslint --cache --fix` + `prettier` / check-only | Lint + format → [config-files](config-files.md) |
| `start` / `start-debug` | build → run `chrome-devtools-mcp.js` | Local server run (debug sets `DEBUG=mcp:*`) |
| `test` / `test:no-build` / `test:only` / `test:update-snapshots` | (build →) `scripts/test.mjs` | Node test runner wrapper → [testing layer](../10-testing/README.md) |
| `prepare` | `scripts/prepare.ts` | Post-install patch of `node_modules` → [build-and-bundle](build-and-bundle.md) |
| `verify-server-json-version` | `scripts/verify-server-json-version.ts` | Checks `server.json` `$schema` vs `mcp-publisher init` → [ci-and-release](ci-and-release.md) |
| `verify-npm-package` | `scripts/verify-npm-package.mjs` | `npm publish --dry-run` asserts key files in tarball → [ci-and-release](ci-and-release.md) |
| `update-lighthouse` | `scripts/update-lighthouse.ts` | Re-vendor Lighthouse bundle + notices → [lighthouse-vendoring](lighthouse-vendoring.md) |
| `eval` | build → `scripts/eval_gemini.ts` | Gemini-driven tool-call eval harness → [eval-and-metrics](eval-and-metrics.md) |
| `count-tokens` | `scripts/count_tokens.ts` | Count Gemini tokens for text/file → [eval-and-metrics](eval-and-metrics.md) |

**Helper modules (not directly npm-invoked):**
- `scripts/append-lighthouse-notices.ts` — concatenates the vendored Lighthouse notices onto the Rollup `THIRD_PARTY_NOTICES` (run as the tail of `bundle`).
- `scripts/eval_result.ts` — `Result`/`TestScenario` types + assertion helpers imported by `eval_gemini.ts` and every scenario.
- `scripts/eslint_rules/*` — custom ESLint rules wired in via `local-plugin.js` → [config-files](config-files.md).
- `scripts/eval_scenarios/*` — eval fixtures, each exporting a `scenario`.
- `scripts/tsconfig.json` — strict NodeNext config for the scripts themselves (`noEmit`, `allowImportingTsExtensions`).

## Relationships
- **Depends on:** the `build/` output (most TS scripts import compiled code); external Gemini API key (`GEMINI_API_KEY`) for eval/count-tokens.
- **Produces / affects:** generated docs/CLI/metrics, the npm tarball, the vendored Lighthouse bundle.

## Gotchas & non-obvious details
- `.ts`/`.mjs` scripts are run with `node` directly (Node strips TS types); `test.mjs` is `.mjs` specifically so it works on Node 20 which lacks `--experimental-strip-types`.
- Generators and metrics scripts import from `build/`, so they require a prior `build`.

## Update triggers
- Any `scripts/*` file is added/removed/renamed, or a `package.json` script is added/rechained.
