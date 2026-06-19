# Eval & Metrics

> **Layer:** Build & tooling · **Sources:** `scripts/eval_gemini.ts`, `scripts/eval_result.ts`, `scripts/eval_scenarios/*`, `scripts/update_metrics.ts`, `scripts/count_tokens.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Three loosely-related developer aids: an LLM-driven eval harness that checks the model picks the right tool calls, a metrics-registry generator that keeps telemetry config in sync with the tools, and a token-counter utility.

## Key files
- `scripts/eval_gemini.ts` — eval runner (`npm run eval`)
- `scripts/eval_result.ts` — `Result`/`TestScenario` types + assertion helpers
- `scripts/eval_scenarios/*_test.ts` — ~19 scenario fixtures, each `export const scenario`
- `scripts/update_metrics.ts` — generates telemetry metrics JSON (`npm run update-metrics`)
- `scripts/count_tokens.ts` — Gemini token counter (`npm run count-tokens`)

## How it works
**Eval harness** (`eval_gemini.ts`, needs `GEMINI_API_KEY`). Boots the built MCP server (`build/src/bin/chrome-devtools-mcp.js`, `--headless` unless `--debug`) over `StdioClientTransport`, exposes it to Gemini via `mcpToTool`, and runs each scenario's prompt with `automaticFunctionCalling` capped at `scenario.maxTurns`. It monkey-patches `client.callTool` to capture every `{name, args}` call, then runs `scenario.expectations(new Result(calls, args))`. A `TestServer` serves any `htmlRoute` (with `<TEST_URL>` substitution) and a random `queryid` is appended to defeat caching. Flags: `--model` (default `gemini-3-flash-preview`), `--debug`, `--repeat` (3×), `--include-skill` (prepend `skills/chrome-devtools/SKILL.md`), `--server-args`. Without positionals it runs every file in `eval_scenarios/`. Exits non-zero on any failure.

**Scenarios.** `eval_result.ts` provides `Result` with `assertNextCall(name, expectedArgs?)`, `consumePageNavigation()`, and `hasPageIdRouting` (detects `--experimental-page-id-routing` in server args). `TestScenario` = `{prompt, maxTurns, expectations, htmlRoute?, serverArgs?}`. Scenarios cover: navigation, console, network, snapshot, input (single/parallel), keyboard focus, fill/select/checkboxes, emulation (viewport/userAgent), isolated context, page-id routing, performance, lighthouse (a11y/best-practices), frontend snapshot, select_page, and an open-ended "fix webpage issues" case. They assert the *sequence/shape of tool calls*, not page outcomes.

**Metrics generator** (`update_metrics.ts`, part of `npm run gen`): imports compiled telemetry helpers and writes two files under `src/telemetry/`:
- `tool_call_metrics.json` — from full + slim `createTools()`; asserts unique tool names, merges with existing via `applyToExistingMetrics` (keeps deprecated entries).
- `flag_usage_metrics.json` — from `cliOptions` via `getPossibleFlagMetrics`, merged with existing.
It also `validateErrorCodes()`: asserts `ErrorCode` enums are sequential from 0 and have no `_<digit>` naming. Fails the run on duplicate names or bad error codes.

**Token counter** (`count_tokens.ts`): `npm run count-tokens -- [-f file] [text]`, calls Gemini `countTokens` (default model `gemini-2.5-flash`); needs `GEMINI_API_KEY`.

## Relationships
- **Depends on:** built server + tools, telemetry modules in [telemetry layer](../07-telemetry/README.md), the `TestServer` from [testing layer](../10-testing/README.md), Gemini API.
- **Produces / affects:** `src/telemetry/{tool_call_metrics,flag_usage_metrics}.json` (regenerated in `gen`, so kept in sync via CI's `check-docs`).

## Gotchas & non-obvious details
- Eval and token-counter need `GEMINI_API_KEY` and are not run in CI; eval is manual.
- `update_metrics.ts` *merges* rather than overwrites — old metrics survive as deprecated, so generated JSON only grows.
- Metrics generation doubles as a validation gate (unique names, error-code ordering).

## Update triggers
- A new eval scenario is added, the metrics registry shape changes, `ErrorCode` enum changes, or the default Gemini model is bumped.
