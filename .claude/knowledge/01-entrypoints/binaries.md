# Binaries (`src/bin/*`)

> **Layer:** Entrypoints & process model · **Sources:** `src/bin/chrome-devtools-mcp.ts`, `src/bin/chrome-devtools-mcp-main.ts`, `src/bin/chrome-devtools-mcp-cli-options.ts`, `src/bin/check-latest-version.ts`, `package.json`, `server.json` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Covers the two npm `bin` commands and their support files. The MCP-server binary (`chrome-devtools-mcp`) is documented here; the human CLI binary (`chrome-devtools`) is in [cli.md](cli.md).

## Key files
- `src/bin/chrome-devtools-mcp.ts` — `#!/usr/bin/env node` shim for the `chrome-devtools-mcp` command.
- `src/bin/chrome-devtools-mcp-main.ts` — the actual MCP server process (imported by the shim).
- `src/bin/chrome-devtools-mcp-cli-options.ts` — `cliOptions` spec + `parseArguments()` (shared with CLI & daemon).
- `src/bin/check-latest-version.ts` — detached npm-version fetcher used by the update check.

## package.json bin map
```jsonc
"bin": {
  "chrome-devtools-mcp": "./build/src/bin/chrome-devtools-mcp.js",
  "chrome-devtools":     "./build/src/bin/chrome-devtools.js"
}
```
`server.json` advertises the package over the **stdio** transport for the MCP registry.

## How it works

### The shim — `chrome-devtools-mcp.ts`
Tiny and runs before anything heavy is loaded:
1. Sets `process.title = 'chrome-devtools-mcp'`.
2. Parses `process.version` and **hard-fails** unsupported Node: errors out for Node `20.x < 20.19`, `22.x < 22.12`, or any `major < 20` (matching `package.json` `engines: ^20.19.0 || ^22.12.0 || >=23`).
3. `await import('./chrome-devtools-mcp-main.js')` — defers loading the real entry until the version gate passes.

### The main — `chrome-devtools-mcp-main.ts`
A top-level-`await` module (it *is* the process body):
1. `import '../polyfill.js'` first (loads bundled polyfills).
2. `await checkForUpdates('Run \`npm install chrome-devtools-mcp@latest\` to update.')`.
3. `args = parseArguments(VERSION)` and, if `args.logFile`, `saveLogsToFile(...)`.
4. Installs an `unhandledRejection` logger (unless `CHROME_DEVTOOLS_MCP_CRASH_ON_UNCAUGHT === 'true'`).
5. Installs `shutdown()` on `stdin` `end`/`close` and on `SIGTERM`/`SIGINT`/`SIGHUP`.
6. `const {server} = await createMcpServer(args, {logFile})`, then `server.connect(new StdioServerTransport())`.
7. `logDisclaimers(args)` and fire-and-forget Clearcut `logDailyActiveIfNeeded()` / `logServerStart(computeFlagUsage(args, cliOptions))`.

**Shutdown semantics:** `shutdown(reason)` is idempotent (`shuttingDown` guard), logs, arms an **unref'd** 10s `setTimeout` that `process.exit(0)` as a backstop if `closeBrowser()` hangs, then awaits `closeBrowser()` and exits 0. The stdin handlers exist because an active Chrome subprocess keeps the event loop ref'd after the MCP client closes the transport, so without them the process would hang.

### Update fetcher — `check-latest-version.ts`
Run as a detached child by `checkForUpdates`. Reads a `cachePath` from `process.argv[2]`, resolves the registry via `npm config get registry` (falling back to `https://registry.npmjs.org`), `fetch`es `<registry>/chrome-devtools-mcp/latest`, and writes `{version}` to the cache. All errors are swallowed.

## `cliOptions` & `parseArguments`
`cliOptions` is a large `satisfies Record<string, YargsOptions>` map — connection (`browserUrl`/`wsEndpoint`/`wsHeaders`/`autoConnect`), launch (`headless`/`executablePath`/`isolated`/`userDataDir`/`channel`/`viewport`/`proxyServer`/`chromeArg`/`ignoreDefaultChromeArg`), category toggles (`category*`), many `experimental*` flags, screenshot defaults, telemetry (`usageStatistics`, `clearcut*`), `slim`, and the hidden `viaCli`. `parseArguments(version, argv?, env?)`:
- yargs-parses `cliOptions` with `scriptName('npx chrome-devtools-mcp@latest')`.
- A **middleware** defaults `channel = 'stable'` when none of `channel/browserUrl/wsEndpoint/executablePath` is set, and force-disables `usageStatistics` when `CI` or `CHROME_DEVTOOLS_MCP_NO_USAGE_STATISTICS` is set.
- Returns `parseSync()`; its type is exported as `ParsedArguments`.

## Relationships
- **Depends on:** `createMcpServer`/`logDisclaimers` ([server-bootstrap.md](server-bootstrap.md)), `closeBrowser` ([../04-browser-cdp/](../04-browser-cdp/)), `ClearcutLogger`/`computeFlagUsage` ([../07-telemetry/](../07-telemetry/)), `StdioServerTransport`/`yargs` (`src/third_party/index.ts`).
- **Used by:** MCP clients spawning the `chrome-devtools-mcp` stdio command; the daemon also spawns this shim as its backing server.

## Gotchas & non-obvious details
- The Node gate is duplicated logic in the shim by design — it must run before `polyfill`/`createMcpServer` imports.
- `viaCli` is hidden and exists only for usage stats; it also affects tool registration (see [server-bootstrap.md](server-bootstrap.md)).
- The shutdown backstop exits with **0** even on a forced timeout — intentional, since the shutdown request was honored.

## Update triggers
- Node version thresholds in the shim or `engines` in `package.json` change.
- A new flag is added to `cliOptions`, or the `parseArguments` middleware defaults change.
- The update-check cache path/registry-resolution logic changes.
