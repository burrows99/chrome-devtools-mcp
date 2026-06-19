# Repository map

> **Layer:** System overview · **Sources:** repo root listing · **Verified at:** commit afa5622 (2026-06-19)

## Top-level directories
| Path | What | Knowledge layer |
|------|------|-----------------|
| `src/` | All first-party TypeScript source | layers 01–08 |
| `src/bin/` | Executable entrypoints (MCP server, CLI) | [01](../01-entrypoints/README.md) |
| `src/daemon/` | Background daemon (client/server) | [01](../01-entrypoints/daemon.md) |
| `src/tools/` | MCP tool definitions, grouped by domain | [02](../02-tools/README.md) |
| `src/devtools/` | CDP connection + vendored-frontend host bindings | [04](../04-browser-cdp/README.md) |
| `src/formatters/` | Token-efficient renderers for collected data | [05](../05-formatters/README.md) |
| `src/trace-processing/` | First-party trace parsing entry | [06](../06-performance-memory/README.md) |
| `src/third_party/` | First-party wrappers around vendored DevTools workers | [06](../06-performance-memory/third-party-workers.md) |
| `src/telemetry/` | Clearcut usage metrics + watchdog | [07](../07-telemetry/README.md) |
| `src/utils/` | Small shared helpers | [08](../08-utilities/README.md) |
| `scripts/` | Build, codegen, vendoring, eval, metrics, test runner | [09](../09-build-tooling/README.md) |
| `tests/` | Test suite (mirrors `src/` layout) | [10](../10-testing/README.md) |
| `skills/` | Shipped agent skills (in npm package) | [11](../11-skills-docs/agent-skills.md) |
| `docs/` | End-user docs (some auto-generated) | [11](../11-skills-docs/docs-reference.md) |
| `.github/` | CI workflows + issue templates | [09](../09-build-tooling/ci-and-release.md) |
| `.claude-plugin/`, `.cursor-plugin/`, `.gemini/` | Editor/agent integrations | [11](../11-skills-docs/integrations.md) |
| `graphify-out/` | Local knowledge-graph build output (git-ignored) | n/a |
| `.claude/` | This knowledge library + project memory | — |

## Notable root files
| File | Purpose | Layer |
|------|---------|-------|
| `package.json` | npm scripts, deps, bin entries | [09](../09-build-tooling/README.md) |
| `tsconfig.json` | TypeScript config | [09](../09-build-tooling/config-files.md) |
| `rollup.config.mjs` | Bundle config (for the vendored bundle) | [09](../09-build-tooling/build-and-bundle.md) |
| `eslint.config.mjs` | Lint config + custom rules | [09](../09-build-tooling/config-files.md) |
| `server.json` | MCP registry manifest | [01](../01-entrypoints/README.md) |
| `release-please-config.json` | Automated release config | [09](../09-build-tooling/ci-and-release.md) |
| `AGENTS.md` | Agent build/test/style instructions | [12](../12-conventions/README.md) |
| `README.md` / `CONTRIBUTING.md` / `SECURITY.md` / `CHANGELOG.md` | Project docs | [11](../11-skills-docs/README.md) |

## Update triggers
- A new top-level directory is added, or `src/` gains a new subsystem directory.
