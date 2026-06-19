# Flag Usage Metrics & On-Disk Persistence

> **Layer:** Telemetry & logging · **Sources:** `src/telemetry/flagUtils.ts`, `src/telemetry/persistence.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
`flagUtils.ts` turns parsed CLI args into a privacy-safe `FlagUsage` payload attached to the `server_start` event (which flags were set, booleans, enum choices — never free-form string values). `persistence.ts` stores a tiny `telemetry_state.json` on disk so the **daily-active** ping fires at most once per UTC day.

## Key files
- `src/telemetry/flagUtils.ts` — `computeFlagUsage` (runtime) + `getPossibleFlagMetrics` (build-time)
- `src/telemetry/persistence.ts` — `Persistence` interface + `FilePersistence` (the `telemetry_state.json` reader/writer)

## Key types & functions
- `computeFlagUsage(args, options)` → `FlagUsage` — runtime, called in CLI main and passed to `logServerStart`.
- `getPossibleFlagMetrics(options)` → `FlagMetric[]` — build-time, enumerates every metric a flag *could* emit (must stay in sync with `computeFlagUsage`).
- `Persistence` — interface: `loadState()`, `saveState(state)`.
- `FilePersistence` — file-backed impl; `LocalState = { lastActive: string }` (ISO-8601 UTC).

## How it works

### Flag usage (`computeFlagUsage`)
Iterates the CLI option definitions. For each flag (name → `snake_case` via `toSnakeCase` then `stripUnderscoreBeforeNumber`):
- **Presence:** logs `${name}_present = boolean` when the flag has *no default*, OR the provided value *differs from* the default (i.e. explicit user intent). Flags left at their default are **not** logged as present.
- **Booleans:** logs the literal value under `${name}`.
- **String enums (`choices`):** logs the value uppercased via `formatEnumChoice` (`${name}_${choice}` → uppercase, e.g. `channel` + `stable` → `CHANNEL_STABLE`).
- Plain string flags (no `choices`) get *only* a `_present` marker — their raw value is never sent.

`getPossibleFlagMetrics` mirrors this for codegen: always emits `${name}_present` (boolean), plus the boolean value or the enum choice list (prepending a `${NAME}_UNSPECIFIED` sentinel). Output is merged into `src/telemetry/flag_usage_metrics.json` by `scripts/update_metrics.ts`. The two functions have explicit "keep in sync" comments.

### Persistence (`FilePersistence`)
State file: `telemetry_state.json` under a platform data dir from `getDataFolder()`:
- macOS → `~/Library/Application Support/chrome-devtools-mcp`
- Windows → `%LOCALAPPDATA%\chrome-devtools-mcp\Data`
- Linux/other → `$XDG_DATA_HOME` or `~/.local/share` + `/chrome-devtools-mcp`

`loadState()` returns `{lastActive: ''}` if the file is missing (a normal first-run case, *not* logged as an error). A read failure that isn't "missing" is logged and reports `ERROR_CODE_PERSISTENCE_FILE_READ_FAILED` via `ClearcutLogger.get()?.logServerError`, then returns the empty default. `saveState()` `mkdir`s recursively and writes pretty-printed JSON; any failure is logged and reports `ERROR_CODE_PERSISTENCE_FILE_SAVE_FAILED` — it never throws (so it can't crash the server). A constructor `dataFolderOverride` exists for tests.

## Relationships
- **Depends on:** `cliOptions` ([CLI options](../01-entrypoints/cli.md)), `transformation.ts` (`stripUnderscoreBeforeNumber`), `utils/string` (`toSnakeCase`), `errors.ts` (`ErrorCode`), `ClearcutLogger` (error reporting — note the import cycle below).
- **Used by:** [CLI main](../01-entrypoints/binaries.md) (`computeFlagUsage` → `logServerStart`), [ClearcutLogger](clearcut-logger.md) (injected `Persistence` for daily-active), `scripts/update_metrics.ts` (`getPossibleFlagMetrics`).

## Gotchas & non-obvious details
- **Only deviations from defaults are logged as present** — telemetry shows *explicit* user configuration, not the full effective config.
- Raw string flag values are never sent (only `_present`); enum flags send the uppercased choice, not arbitrary text.
- `persistence.ts` imports `ClearcutLogger` while `ClearcutLogger` holds a `Persistence` — a real but benign cyclic dependency; persistence uses the lazy `ClearcutLogger.get()` accessor.
- The state file holds a single field (`lastActive`); there is no device ID or counter on disk.
- `getDataFolder()` honors `XDG_DATA_HOME` / `LOCALAPPDATA` env overrides.

## Update triggers
- A new CLI flag (verify `computeFlagUsage` ↔ `getPossibleFlagMetrics` stay in sync) or a new flag type.
- The on-disk state shape, file name, or data-folder logic changes.
- A new persistence-related `ErrorCode`.
