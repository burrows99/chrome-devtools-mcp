# Contributing & release process

> **Layer:** Conventions · **Sources:** `CONTRIBUTING.md`, `.github/workflows/*`, `release-please-config.json` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The workflow rules for landing changes and shipping releases. CI-level detail
lives in [ci-and-release](../09-build-tooling/ci-and-release.md); this is the
contributor-facing summary.

## Before contributing
- A **Google CLA** is required (see `CONTRIBUTING.md`).
- All submissions go through **GitHub PR review**.

## Commit & PR conventions
- **Conventional Commits** for PR/commit titles (validated by the
  `conventional-commit` workflow).
- Use **`chore:`** for incomplete features not yet user-visible. When the feature
  is ready, open a separate **`feat:`** PR that enables it.

## Feature release checklist (from `CONTRIBUTING.md`)
A `feat:` is ready only when:
- Docs are updated (`README.md`, tool reference) — run **`npm run gen`**.
- It works on Chrome stable, or version restrictions are documented.
- Corresponding [skills](../11-skills-docs/agent-skills.md) are added/updated.
- It actually fulfills a use case end to end (no orphan tools).

## Release flow
- Automated by **release-please** + GitHub Actions. A `chore(main): release
  chrome-devtools-mcp` PR is opened automatically when changelog-worthy changes
  land on `main`; review, test, and merge it to publish.
- Publishing pushes to **npm** and the **MCP registry** (`server.json` version is
  verified by `scripts/verify-server-json-version.ts`).

## Updating the vendored Lighthouse dependency
1. Bump the Lighthouse version in `package.json`, `npm install`.
2. Check out the matching Lighthouse repo revision to `../lighthouse` (sibling dir).
3. Run **`npm run update-lighthouse`** (requires yarn).
4. Commit the bundle; if new deps appear, update `tests/third_party_notices.test.ts`.

Detail: [lighthouse-vendoring](../09-build-tooling/lighthouse-vendoring.md).

## Local dev commands
```sh
npm ci && npm run build      # install + build
npm run test                 # build then run all tests
npm run test tests/X.test.ts # single test file
npm run format               # fix lint + formatting
npm run gen                  # regenerate docs/CLI/metrics after tool changes
```

## Update triggers
- The release process, commit policy, or Lighthouse-update steps change in
  `CONTRIBUTING.md` / workflows.
