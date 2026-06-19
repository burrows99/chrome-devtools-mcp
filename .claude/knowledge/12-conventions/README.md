# 12 · Conventions

> **Layer:** Conventions · **Sources:** `AGENTS.md`, `CONTRIBUTING.md`, `docs/design-principles.md`, `eslint.config.mjs` · **Verified at:** commit afa5622 (2026-06-19)

The rules and principles every change to this repo must respect. These are
**normative** — CI and custom ESLint rules enforce much of it.

## Read
- [design-principles.md](./design-principles.md) — the product/API philosophy tools must follow
- [typescript-rules.md](./typescript-rules.md) — hard TypeScript constraints (no `any`, no `as`, …) and JSON-schema restrictions
- [contributing-and-release.md](./contributing-and-release.md) — commits, feature checklist, release flow, Lighthouse update procedure

## The short version
- Run everything through **`package.json` scripts** (`npm run build`, `npm run test`, `npm run format`). See [build-tooling](../09-build-tooling/README.md).
- After adding/renaming a tool, run **`npm run gen`** to regenerate docs/CLI. See [code-generation](../09-build-tooling/code-generation.md).
- **Conventional commits** for PR/commit titles; `chore:` for not-yet-released features, `feat:` to flip them on.
- Tool design follows the [design principles](./design-principles.md): agent-agnostic, token-optimized, small deterministic blocks, self-healing errors, progressive complexity, reference-over-value.

## Update triggers
- `AGENTS.md`, `CONTRIBUTING.md`, the design principles, or the lint ruleset change.
