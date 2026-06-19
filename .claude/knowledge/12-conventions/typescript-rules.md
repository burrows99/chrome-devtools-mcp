# TypeScript & schema rules

> **Layer:** Conventions · **Sources:** `AGENTS.md`, `CONTRIBUTING.md`, `eslint.config.mjs`, `scripts/eslint_rules/` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Hard constraints on TypeScript and Zod schemas in this repo. Several are enforced
by custom ESLint rules, so violations fail CI (`npm run check-format`).

## TypeScript rules (from `AGENTS.md`)
- **No `any`** type.
- **No `as`** type casting.
- **No `!`** non-null assertion operator.
- **No `// @ts-ignore`, `// @ts-nocheck`, `// @ts-expect-error`** comments.
- Prefer **`for..of`** over `Array.prototype.forEach`.

These push you toward proper type narrowing and runtime guards instead of
escape hatches. When the type system fights you, model the type correctly rather
than reaching for `as`/`!`.

## JSON / Zod schema restrictions (from `CONTRIBUTING.md`)
Tool argument schemas are constrained so they translate cleanly to MCP/JSON Schema
and to CLI options:
- **No `.nullable()`** and **no `.object()`** types — enforced by the
  `@local/enforce-zod-schema` ESLint rule (`scripts/eslint_rules/`).
- Represent a complex object as a **short formatted string** parameter instead.

See how schemas are declared in [tool-definition](../02-tools/tool-definition.md)
and consumed by codegen in [code-generation](../09-build-tooling/code-generation.md).

## Other enforced rules
- A **license-header** rule (`scripts/eslint_rules/`) requires the Apache-2.0
  header on source files.
- Formatting via `prettier` (`.prettierrc.cjs`). Run `npm run format` to auto-fix
  lint + formatting.

## Commands
```sh
npm run format        # eslint --fix + prettier --write
npm run check-format  # eslint + prettier --check (CI gate)
npm run typecheck     # tsc --noEmit
```

## Update triggers
- A rule is added/removed in `AGENTS.md` or `scripts/eslint_rules/`, or the
  schema restrictions change.
