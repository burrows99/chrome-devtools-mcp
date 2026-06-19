# The `chrome-devtools` CLI

> **Layer:** Entrypoints & process model · **Sources:** `src/bin/chrome-devtools.ts`, `src/bin/chrome-devtools-cli-options.ts`, `src/bin/chrome-devtools-mcp-cli-options.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`chrome-devtools` is the human-facing CLI. Unlike the MCP server (which speaks the protocol over stdio), the CLI exposes `start`/`stop`/`status` plus **one subcommand per MCP tool**, each of which runs synchronously against a shared background [daemon](daemon.md) and prints a formatted result.

## Key files
- `src/bin/chrome-devtools.ts` — `#!/usr/bin/env node` shim + the whole yargs command tree.
- `src/bin/chrome-devtools-cli-options.ts` — **auto-generated** `commands` map (`npm run cli:generate`); one `ArgDef` set per tool.
- `src/bin/chrome-devtools-mcp-cli-options.ts` — shared `cliOptions`/`parseArguments`, reused for the `start` command's flags.

## Key types & functions
- `commands: Commands` — `Record<toolName, {description, category, args: Record<name, ArgDef>}>` (generated).
- `ArgDef` — `{name, type, description, required, default?, enum?}` per tool argument.
- `start(args, sessionId)` — appends `defaultArgs`, `startDaemon(...)`, then `logDisclaimers(parseArguments(...))`.

## How it works

### Shim & global yargs setup
`process.title = 'chrome-devtools'`, then `await checkForUpdates('Run \`npm install -g chrome-devtools-mcp@latest\` ...')`. The yargs instance is built with `scriptName('chrome-devtools')`, `.demandCommand()`, `.strict()`, `.version(VERSION)`, and a hidden global `--sessionId` (default `''`) used to scope the daemon.

### CLI-specific defaults
`defaultArgs = ['--viaCli', '--experimentalStructuredContent']` are appended to every daemon start. `startCliOptions` clones `cliOptions` and tweaks them for humans:
- deletes `viewport` (no CLI serialization), `experimentalStructuredContent`, `experimentalInteropTools`, `experimentalPageIdRouting`;
- `headless` default → `true`; `categoryExtensions` default → `true`;
- rewrites the `isolated` description (defaults true unless `userDataDir` given).
Guard `throw`s assert that `headless` still has a default and `isolated` still has none (so the manual default logic below stays correct).

### Built-in commands
- **`start`** — if a daemon for this `sessionId` is running, `stopDaemon` first (restart). Manually defaults `isolated = true` (when neither `isolated` nor `userDataDir` set) and `headless = true` so yargs conflict resolution isn't disturbed. `serializeArgs(cliOptions, argv)` round-trips flags to strings, then `start(args, sessionId)`; exits 0.
- **`status`** — if running, prints a line and `sendCommand({method:'status'})`, then parses+prints `pid/socket/start-date/version/args`; else prints "not running". Exits 0.
- **`stop`** — `stopDaemon(sessionId)` if running; exits 0.

### Per-tool subcommands (generated loop)
For each `[commandName, commandDef]` in `commands`, the CLI builds a yargs command string: required args become positionals `<arg>`, optional args become `[--arg]`. The builder adds a global `--output-format` (`md`|`json`, default `md`) and maps each `ArgDef.type` (`integer`/`number`→`number`, `boolean`, `array`, else `string`) to a yargs positional/option, carrying `default`/`enum`→`choices`.

The handler:
1. If no daemon running, `start([], sessionId)` (auto-starts with defaults).
2. Collects only the args present in `argv` into `commandArgs`.
3. `sendCommand({method:'invoke_tool', tool: commandName, args: commandArgs}, sessionId)`.
4. On success, `console.log(await handleResponse(JSON.parse(response.result), outputFormat))`; on failure prints `Error:` and exits 1. Any throw → "Failed to execute command" + exit 1.

`handleResponse` (from `daemon/client.ts`) flattens `CallToolResult` content: text chunks are joined; **image content is written to a temp file** and replaced with `Saved to <path>.`. With `--output-format json` it prefers `structuredContent` (plus an `images` array), falling back to text for back-compat.

## Relationships
- **Depends on:** the daemon client ([daemon.md](daemon.md) — `startDaemon`/`stopDaemon`/`sendCommand`/`handleResponse`, `isDaemonRunning`/`serializeArgs`), `logDisclaimers` ([server-bootstrap.md](server-bootstrap.md)), shared `cliOptions`/`parseArguments` ([binaries.md](binaries.md)), and `yargs`/`CallToolResult` (`src/third_party/index.ts`).
- **Used by:** humans on the command line. Generated from the tool layer ([../02-tools/](../02-tools/)).

## Gotchas & non-obvious details
- `chrome-devtools-cli-options.ts` is **auto-generated — do not hand-edit**; regenerate with `npm run cli:generate` when tools change.
- The CLI always passes `--viaCli`, which keeps disabled-category tools *registered* on the server so their subcommands return a helpful enable-me error instead of failing as unknown.
- The CLI's defaults differ from the MCP server's: `headless` and `categoryExtensions` default **true** here.
- `--sessionId` scopes the daemon's socket/PID paths, enabling multiple independent daemons per user.

## Update triggers
- The set of MCP tools changes (regenerate `chrome-devtools-cli-options.ts`).
- `defaultArgs`, the `startCliOptions` overrides, or the start/stop/status command behavior changes.
- `handleResponse` output formatting changes.
