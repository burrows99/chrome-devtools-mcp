# Config Files & Lint Rules

> **Layer:** Build & tooling · **Sources:** `tsconfig.json`, `scripts/tsconfig.json`, `eslint.config.mjs`, `scripts/eslint_rules/*`, `.prettierrc.cjs`, `.prettierignore`, `.nvmrc` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Defines the TypeScript compiler settings, the flat-config ESLint setup (including three custom rules), and Prettier formatting that all code must satisfy. `check-format` enforces these in CI.

## Key files
- `tsconfig.json` — app/test build config (target es2023, `moduleResolution: bundler`, `outDir build`)
- `scripts/tsconfig.json` — stricter NodeNext config for scripts (`strict`, `noEmit`, `allowImportingTsExtensions`)
- `eslint.config.mjs` — flat config combining `js`, `typescript-eslint`, `@stylistic`, `import`, `@local`
- `scripts/eslint_rules/{local-plugin,check-license-rule,enforce-zod-schema-rule,no-direct-third-party-imports-rule}.js`
- `.prettierrc.cjs`, `.prettierignore`, `.nvmrc` (`v24`)

## How it works
**TypeScript.** Root `tsconfig.json` targets `es2023`, uses `moduleResolution: bundler`, `allowJs`, `incremental`, `sourceMap`, and `useUnknownInCatchVariables: false`. Its `include` pulls in a large explicit list of `chrome-devtools-frontend/front_end/...` directories (so they're typechecked/compiled alongside `src` and `tests`) while excluding `*.test.ts`. `scripts/tsconfig.json` extends it but flips to `module/moduleResolution: nodenext`, `strict: true`, `noEmit`, `allowImportingTsExtensions` (scripts import `./eval_result.ts` with extension).

**ESLint** (flat config): globally ignores `node_modules`, `build/`, test fixtures, and the Lighthouse vendored bundle. Extends `js/recommended`, `tseslint.configs.recommended` + `stylistic`, and `importPlugin` typescript config. Notable enforced rules:
- `@local/check-license` (error, all files) — license header.
- `curly: all`, `@stylistic/semi`, `@stylistic/function-call-spacing`.
- `@typescript-eslint`: `consistent-type-imports`/`consistent-type-exports`, `consistent-type-definitions: interface`, `array-type: array-simple`, `no-floating-promises`, `no-explicit-any` (allow rest args), `no-unused-vars` ignoring `^_`.
- `import/order` (alphabetized, newlines-between), `import/no-cycle` (maxDepth Infinity), `import/enforce-node-protocol-usage: always` (forces `node:` prefix).
- `no-restricted-imports` blocks deep `chrome-devtools-frontend/*` paths — only `mcp/mcp.js` is allowed.
- Scoped configs: `@local/no-direct-third-party-imports` on `src/**/*.ts`; `@local/enforce-zod-schema` on `src/tools/**/*.ts`; `no-floating-promises` off in `*.test.ts`.

**Custom rules** (registered in `local-plugin.js`):
- `check-license` — fixable; inserts the `@license Copyright <year> Google LLC / SPDX Apache-2.0` block (and a blank line after it) if missing. Skips `.json`. Honors shebangs.
- `enforce-zod-schema` — bans `.nullable()` (use `.optional()`) and `z.object()`/`zod.object()` (represent complex objects as a formatted string) in tool schema files.
- `no-direct-third-party-imports` — disallows *value* imports of bundled packages outside `src/third_party/`; package list is discovered at lint-load time by scanning `src/third_party/*.ts` for imports. Type-only imports are allowed. Prevents dev-only imports that break after bundling (issue #1123).

**Prettier:** `bracketSpacing:false`, `singleQuote:true`, `trailingComma:all`, `arrowParens:avoid`, `endOfLine:lf`. Ignores `CHANGELOG.md`, the Lighthouse bundle, and the release-please-managed plugin JSON files (their formatting would break CI).

## Relationships
- **Depends on:** `src/third_party/index.ts` (drives the no-direct-import allow-list) and the tool-schema convention in [tools layer](../02-tools/README.md).
- **Produces / affects:** every source file's shape; enforced by `format`/`check-format` and the `presubmit.yml` `check-format` gate → [ci-and-release](ci-and-release.md).

## Gotchas & non-obvious details
- `.nvmrc` says `v24` but `run-tests.yml` builds on Node 22 and tests on 22/24/26 — the build only needs 22+.
- The license header year is `new Date().getFullYear()` — running the autofixer in a new year rewrites headers.
- `enforce-zod-schema` is a hard wall: tool params can't be nested objects.

## Update triggers
- TS target/module settings change, an ESLint rule is added/relaxed, a custom rule's logic changes, or Prettier options change.
