# 00 · Overview

> **Layer:** System overview · **Sources:** `README.md`, `docs/design-principles.md`, `package.json` · **Verified at:** commit afa5622 (2026-06-19)

Start here. This layer gives the 10,000-foot view before you descend into the
implementation layers.

## Read in this order
1. [what-is-this.md](./what-is-this.md) — what the product is and who uses it
2. [architecture.md](./architecture.md) — the layered architecture + the lifecycle of one tool call
3. [repository-map.md](./repository-map.md) — every top-level directory, one line each
4. [tech-stack.md](./tech-stack.md) — runtime, key dependencies, why each is here
5. [glossary.md](./glossary.md) — MCP, CDP, Puppeteer, Lighthouse, Clearcut, UID, etc.

## One-paragraph summary
`chrome-devtools-mcp` is a **Model Context Protocol (MCP) server** (plus a
standalone CLI) that gives AI coding agents control of a **live Chrome browser**.
It wraps **Puppeteer** for automation and the **Chrome DevTools Protocol (CDP)** +
a vendored slice of **Chrome DevTools Frontend / Lighthouse** for inspection and
performance analysis. Agents call **tools** (navigate, click, screenshot, record a
performance trace, take a heap snapshot, read console/network, …); the server runs
them against Chrome and returns **token-optimized** text/structured responses.

## Layer map (high → low)
| # | Layer | Dir |
|---|-------|-----|
| 00 | Overview | [00-overview](.) |
| 01 | Entrypoints & process model | [01-entrypoints](../01-entrypoints/README.md) |
| 02 | Tool system | [02-tools](../02-tools/README.md) |
| 03 | Context & state | [03-context-state](../03-context-state/README.md) |
| 04 | Browser & CDP | [04-browser-cdp](../04-browser-cdp/README.md) |
| 05 | Formatters | [05-formatters](../05-formatters/README.md) |
| 06 | Performance & memory engine | [06-performance-memory](../06-performance-memory/README.md) |
| 07 | Telemetry & logging | [07-telemetry](../07-telemetry/README.md) |
| 08 | Utilities & shared types | [08-utilities](../08-utilities/README.md) |
| 09 | Build & tooling | [09-build-tooling](../09-build-tooling/README.md) |
| 10 | Testing | [10-testing](../10-testing/README.md) |
| 11 | Skills, docs & integrations | [11-skills-docs](../11-skills-docs/README.md) |
| 12 | Conventions | [12-conventions](../12-conventions/README.md) |

See the [knowledge-library index](../README.md) for how to maintain these docs.
