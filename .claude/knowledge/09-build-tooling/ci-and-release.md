# CI & Release

> **Layer:** Build & tooling · **Sources:** `.github/workflows/*.yml`, `release-please-config.json`, `.release-please-manifest.json`, `server.json`, `scripts/verify-server-json-version.ts`, `scripts/verify-npm-package.mjs` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The CI gates that protect `main`, plus the fully automated release flow: conventional commits → release-please PR → version bump → git tag → npm + MCP registry publish.

## Key files
- `.github/workflows/presubmit.yml` — `check-format` + `check-docs` gates
- `.github/workflows/run-tests.yml` — OS × Node test matrix + `test-success` gate
- `.github/workflows/conventional-commit.yml` — validates PR titles
- `.github/workflows/release-please.yml` — opens/maintains the release PR
- `.github/workflows/pre-release.yml` — verifies artifacts on `release-please-*` branches
- `.github/workflows/publish-to-npm-on-tag.yml` — publishes on tag push
- `release-please-config.json` / `.release-please-manifest.json` — changelog sections + version-stamped files

## How it works
**Presubmit gates** (PR, push to main, merge_group):
- `check-format` — `npm ci` then `npm run check-format` (eslint + prettier --check).
- `check-docs` — runs `npm run gen` and fails if `git diff` is non-empty, i.e. generated docs/CLI/metrics must already be committed. This is the enforcement arm of [code-generation](code-generation.md)/[eval-and-metrics](eval-and-metrics.md).

**Tests** (`run-tests.yml`): matrix of `{ubuntu, windows, macos} × node {22,24,26}`, `fail-fast:false`. Builds once on Node 22 with `npm run bundle` (`--max_old_space_size=4096`), installs Chrome via puppeteer, then re-pins Node to the matrix version and runs `npm run test:no-build` (Windows uses `--test-concurrency=1`; merge_group adds `--retry`). Ubuntu disables AppArmor's unprivileged userns restriction. A `test-success` job is the required branch-protection gate.

**Conventional commits** (`conventional-commit.yml`): on `pull_request_target` (and merge_group), validates the PR title with `action-semantic-pull-request`. Commit types map to changelog sections in `release-please-config.json` (`feat`→Features, `fix`→Fixes, `docs`→Documentation, `perf`/`refactor` shown; `chore`/`test`/`build`/`ci` hidden).

**Release flow:**
1. `release-please.yml` (push to main) runs `release-please-action@v5` with the config + manifest, opening/updating a release PR that bumps the version across `extra-files`: `src/version.ts` (via `x-release-please` markers), `server.json` (×2 jsonpaths), the `.claude-plugin`/`.cursor-plugin`/`.github/plugin`/`gemini-extension.json` files (version + the MCP `args[0]`). Uses `BROWSER_AUTOMATION_BOT_TOKEN`.
2. On the `release-please-*` branch, `pre-release.yml` builds the bundle and runs `verify-server-json-version` (downloads `mcp-publisher`, runs `init`, asserts `$schema` matches) and `verify-npm-package` (`npm publish --dry-run --json`, asserts `build/src/index.js` and `build/src/third_party/index.js` are in the tarball).
3. Merging the release PR creates a `chrome-devtools-mcp-v*` tag → `publish-to-npm-on-tag.yml` runs: builds with `NODE_ENV=production`, `npm publish --provenance --access public` for `chrome-devtools-mcp`, then rewrites `package.json#name` to `chrome-devtools` (+ a stub README) and publishes that alias too. A dependent job installs `mcp-publisher`, logs in via `github-oidc`, and `mcp-publisher publish`es to the MCP registry. Both publish jobs are also `workflow_dispatch`-able with toggles.

## Relationships
- **Depends on:** `npm run gen`/`bundle` (build & generation layers), `server.json`/version files, GitHub OIDC + bot token secrets.
- **Produces / affects:** the published `chrome-devtools-mcp` + `chrome-devtools` npm packages and the MCP registry entry consumed by users of [entrypoints](../01-entrypoints/README.md).

## Gotchas & non-obvious details
- `check-docs` means forgetting `npm run gen` after a tool change fails CI even when code compiles.
- Two npm packages ship from one tag: `chrome-devtools-mcp` and the alias `chrome-devtools` (name swapped in-job).
- `verify-npm-package` only asserts a *subset* of tarball paths; it's a smoke check, not exhaustive.
- Version is the single source of truth in `.release-please-manifest.json` (`1.3.0`) and fanned out by `release-please-config.json` extra-files.

## Update triggers
- A workflow file changes, a new version-stamped file is added to `release-please-config.json`, the publish targets change, or branch-protection gate names change.
