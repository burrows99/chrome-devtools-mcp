# Console Formatter

> **Layer:** Formatters · **Sources:** `src/formatters/ConsoleFormatter.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Formats Puppeteer/CDP `ConsoleMessage`s and `UncaughtError`s into concise list rows or detailed single-message output (resolved arguments + symbolized stack trace). Also collapses runs of identical messages, mirroring Chrome DevTools' console grouping.

## Key files
- `ConsoleFormatter.ts` — the formatter, the `GroupedConsoleFormatter` subclass, and the text-conversion free functions.

## Key types & functions
- `ConsoleFormatter` — main class; `protected` constructor, built via the async factory.
- `ConsoleFormatter.from(msg, options)` — static async factory. Resolves args/stack only when `options.fetchDetailedData` is set; handles `UncaughtError` vs `ConsoleMessage` separately.
- `ConsoleFormatter.groupConsecutive(messages)` — static; collapses adjacent same-`type`/`text`/`argCount` `ConsoleFormatter`s into `GroupedConsoleFormatter`s with a `count`. Accepts and passes through `IssueFormatter`s untouched.
- `GroupedConsoleFormatter` — subclass that adds `count` to the concise JSON and renders a ` [N times]` suffix.
- `ConsoleMessageConcise` / `ConsoleMessageDetailed` — internal JSON shapes (detailed extends concise, adding pre-formatted `args: string[]` and `stackTrace?: string`).
- `ConsoleFormatterOptions` — `{id, fetchDetailedData?, devTools?, …ForTesting}`. The `*ForTesting` fields inject resolved args/stack/cause/ignore-check so tests skip CDP.
- `IgnoreCheck` — predicate `(frame) => boolean`; frames it matches are dropped from stack traces.

## How it works
**Concise** (`toString()` → `convertConsoleMessageConciseToString`):
```
msgid=<id> [<type>] <text> (<argsCount> args)[ [<N> times]]
```
**Detailed** (`toStringDetailed()` → `convertConsoleMessageConciseDetailedToString`): `ID:`, `Message: <type>> <text>`, then optional `### Arguments` (`Arg #i: …`) and `### Stack trace` sections, newline-joined with empty parts filtered.

**Detail fetching** (`from`, gated by `fetchDetailedData`):
- `UncaughtError` → `SymbolizedError.fromDetails(...)` (`includeStackAndCause` mirrors `fetchDetailedData`); message/stack/cause stored.
- `ConsoleMessage` → each arg resolved via `arg.jsonValue()`, except error-typed remote objects which go through `SymbolizedError.fromError(...)`. Failures fall back to `<error: Argument i is no longer available>`.
- Stack trace built via `createStackTraceForConsoleMessage(devTools, msg)` (best-effort; errors ignored).

**Arg/text coupling** (`#getArgs`): if `text` is empty, the first resolved arg *is* the text, so it is shifted off before formatting (mirrors DevTools `formatMessage`). `formatArg` JSON-stringifies objects, `String()`s primitives, and recursively formats nested `SymbolizedError`s.

## Relationships
- **Depends on:** `SymbolizedError`, `createStackTraceForConsoleMessage`, `TargetUniverse` from [`src/devtools/DevtoolsUtils.ts`](../04-browser-cdp/); `UncaughtError` from `src/PageCollector.js` ([collectors](../03-context-state/)); DevTools `StackTrace` model + `IgnoreListManager` via `src/third_party/index.js`; `IssueFormatter` (type-only, for `groupConsecutive`).
- **Used by:** [`McpResponse`](../03-context-state/) — list path uses `from({fetchDetailedData:false})` + `groupConsecutive` + `toString()`/`toJSON()`; drill-down uses `from({fetchDetailedData:true})` + `toStringDetailed()`/`toJSONDetailed()`.

## Gotchas & non-obvious details
- Stack output capped at `STACK_TRACE_MAX_LINES = 50`, then `... and N more frames`; always ends with `Note: line and column numbers use 1-based indexing`.
- Ignore-listed frames are filtered *per fragment*; an async fragment whose frames are all ignored emits no `--- <description> ---` separator at all.
- `groupConsecutive` only groups **consecutive** items and only `ConsoleFormatter`s — an interleaved `IssueFormatter` breaks a run.
- `argCount` falls back to `msg.args().length` when no resolved args (concise path), so the count is correct even without detail fetching.

## Update triggers
- Concise/detailed text format strings change.
- Grouping key (`type`/`text`/`argCount`) or `[N times]` rendering changes.
- `STACK_TRACE_MAX_LINES` or the 1-based-indexing note changes.
- New `ConsoleFormatterOptions` fields or arg-resolution behavior changes.
