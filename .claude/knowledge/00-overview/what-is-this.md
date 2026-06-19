# What is `chrome-devtools-mcp`?

> **Layer:** System overview · **Sources:** `README.md`, `package.json`, `server.json` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`chrome-devtools-mcp` ("Chrome DevTools for agents") lets a coding agent
(Antigravity, Claude, Cursor, Copilot, Gemini, …) **control and inspect a live
Chrome browser**. It is published to npm as `chrome-devtools-mcp` and to the MCP
registry as `io.github.ChromeDevTools/chrome-devtools-mcp`. Maintained by Google
LLC under Apache-2.0; the canonical repo is `ChromeDevTools/chrome-devtools-mcp`.

## Two products, one codebase
- **MCP server** — `chrome-devtools-mcp` binary. Speaks MCP over stdio to an MCP
  client. This is the primary product. See [server bootstrap](../01-entrypoints/server-bootstrap.md).
- **CLI** — `chrome-devtools` binary. Same capabilities without an MCP client, for
  scripting/manual use. See [cli](../01-entrypoints/cli.md). Docs: `docs/cli.md`.

## Key features (from README)
- **Performance insights** — records traces via the vendored DevTools/Lighthouse
  engine and extracts actionable insights (LCP, CLS, …) rather than raw JSON.
- **Advanced debugging** — network requests, screenshots, console messages with
  source-mapped stack traces, accessibility snapshots, heap snapshots.
- **Reliable automation** — Puppeteer drives Chrome and auto-waits for results.

## How agents consume it
1. The agent's MCP client launches the server (`npx chrome-devtools-mcp@latest`).
2. The server advertises a set of **tools** (see [tool system](../02-tools/README.md)).
3. The agent calls tools; the server runs them against Chrome and returns
   token-optimized output (see [design principles](../12-conventions/design-principles.md)).
4. A **`--slim`** mode exposes a reduced tool set for basic tasks
   (see [slim-mode](../02-tools/slim-mode.md)).

## Important product facts
- Officially supports **Google Chrome** and **Chrome for Testing** only.
- **Usage statistics are collected by default** (opt out with `--no-usage-statistics`
  or `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS`/`CI`). See [telemetry](../07-telemetry/README.md).
- Performance tools may send trace URLs to the **CrUX API** (opt out with
  `--no-performance-crux`).
- Periodically checks npm for updates (disable with
  `CHROME_DEVTOOLS_MCP_NO_UPDATE_CHECKS`).
- Exposes browser contents to the MCP client — a stated security/privacy caveat.

## Update triggers
- The product pitch, supported-browser policy, or privacy/opt-out flags change in `README.md`.
