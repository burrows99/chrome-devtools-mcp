# Tech stack

> **Layer:** System overview · **Sources:** `package.json`, `.nvmrc`, `tsconfig.json` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
What the project runs on and the dependencies that shape its architecture.

## Runtime & language
- **Node.js** `^20.19.0 || ^22.12.0 || >=23` (see `engines`; `.nvmrc` pins a dev version).
- **TypeScript** `^6.x`, ESM (`"type": "module"`). Compiled with `tsc`; allows
  importing `.ts` extensions. Strict style rules — see
  [typescript-rules](../12-conventions/typescript-rules.md).

## Core dependencies (and why)
| Dependency | Role |
|------------|------|
| `@modelcontextprotocol/sdk` | MCP server/protocol implementation — the agent interface |
| `puppeteer` (+ `puppeteer-core` override) | Drives Chrome, auto-waits for actions → [04](../04-browser-cdp/README.md) |
| `chrome-devtools-frontend` | Source of the vendored DevTools/trace engine bundle → [06](../06-performance-memory/README.md) |
| `lighthouse` | Performance analysis (used for types + vendored bundle) |
| `zod` (via MCP SDK) | Tool argument schemas → [tool-definition](../02-tools/tool-definition.md) |
| `yargs` | CLI/flag parsing → [01](../01-entrypoints/cli.md) |
| `@google/genai` | Gemini client for evals (`scripts/eval_gemini.ts`) |
| `@toon-format/toon` | Compact serialization format used in output |
| `debug` | Namespaced logging → [logging](../07-telemetry/logging.md) |
| `semver` | Version checks / update notifications |

## Dev / build tooling
- **Build**: `tsc` + `scripts/post-build.ts`; **bundle**: `rollup` + `rollup-plugin-license`.
- **Lint/format**: `eslint` (+ `typescript-eslint`, `@stylistic`, custom
  `scripts/eslint_rules`) and `prettier`.
- **Test**: Node's built-in `node:test` via `scripts/test.mjs`; `sinon` for stubs;
  `.js.snapshot` snapshot files. → [10](../10-testing/README.md)
- **Release**: `release-please` + GitHub Actions. → [ci-and-release](../09-build-tooling/ci-and-release.md)

## Update triggers
- A major dependency is added/removed/bumped, or the Node/TypeScript baseline changes.
