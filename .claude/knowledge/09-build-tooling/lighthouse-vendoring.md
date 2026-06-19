# Lighthouse Vendoring

> **Layer:** Build & tooling · **Sources:** `scripts/update-lighthouse.ts`, `scripts/append-lighthouse-notices.ts`, `src/third_party/lighthouse-devtools-mcp-bundle.js`, `src/third_party/index.ts`, `rollup.config.mjs` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The Lighthouse audit engine is vendored as a single committed bundle (`src/third_party/lighthouse-devtools-mcp-bundle.js`, ~8 MB) plus its third-party license notices. This doc covers how that bundle is (re)produced and how its licenses end up in the shipped `THIRD_PARTY_NOTICES`.

## Key files
- `scripts/update-lighthouse.ts` — re-vendors the bundle from a sibling Lighthouse checkout (`npm run update-lighthouse`)
- `scripts/append-lighthouse-notices.ts` — appends Lighthouse notices to the Rollup notices (tail of `npm run bundle`)
- `src/third_party/lighthouse-devtools-mcp-bundle.js` — the committed bundle (checked in, not built locally)
- `src/third_party/LIGHTHOUSE_MCP_BUNDLE_THIRD_PARTY_NOTICES` — committed notices for that bundle
- `src/third_party/index.ts` — re-exports `snapshot`, `navigation`, `generateReport` from the bundle

## How it works
**Re-vendoring** (`update-lighthouse.ts`, run manually): expects a `lighthouse` git checkout as a sibling of the repo (`../lighthouse`). It runs `yarn` then `yarn build-devtools-mcp` in that checkout, then copies `dist/lighthouse-devtools-mcp-bundle.js` and `dist/LIGHTHOUSE_MCP_BUNDLE_THIRD_PARTY_NOTICES` into `src/third_party/`. The bundle is committed to the repo — normal builds never regenerate it.

**Consumption.** `src/third_party/index.ts` imports `snapshot`/`navigation`/`generateReport` from the bundle and re-exports them with typed wrappers (using Lighthouse's `Flags`/`Result`/`RunnerResult` types). This is what the Performance/Lighthouse tools call.

**Notices during bundle.** Rollup's `license` plugin writes a `THIRD_PARTY_NOTICES` for everything *it* inlines (allow-listed licenses only; manual handling adds `chrome-devtools-frontend` and `lighthouse` LICENSE files and the front_end third_party LICENSEs, joined by a `DEPENDENCY DIVIDER`). Because the Lighthouse bundle is pre-built (not run through Rollup's resolver), its own dependencies aren't captured there — so `append-lighthouse-notices.ts` reads `src/third_party/LIGHTHOUSE_MCP_BUNDLE_THIRD_PARTY_NOTICES` and appends it onto `build/src/third_party/THIRD_PARTY_NOTICES` with the same divider. This runs as the last step of `npm run bundle`.

## Relationships
- **Depends on:** an out-of-repo `../lighthouse` checkout (only for re-vendoring); the Rollup license output from [build-and-bundle](build-and-bundle.md).
- **Produces / affects:** the Lighthouse functions used by [Performance & memory](../06-performance-memory/README.md); the final `THIRD_PARTY_NOTICES` shipped in the package.

## Gotchas & non-obvious details
- The bundle is a committed ~8 MB artifact, deliberately excluded from ESLint and Prettier (see [config-files](config-files.md)).
- Re-vendoring is a manual, off-CI process and requires a *sibling* Lighthouse repo built with `yarn build-devtools-mcp`.
- Lighthouse's transitive license notices are only complete because of the manual `append-lighthouse-notices.ts` step — skipping it would ship incomplete attribution.

## Update triggers
- Lighthouse is re-vendored (new `lighthouse` version) → both the bundle and its notices file change.
- The Rollup license template or the divider string changes.
