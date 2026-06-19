# Trace Processing (`parse.ts`)

> **Layer:** Performance & memory engine · **Sources:** `src/trace-processing/parse.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Parse a raw recorded performance-trace buffer into a vendored `ParsedTrace` plus its `insights`, and format that data into human/LLM-readable summaries and per-insight detail. This file is first-party glue; the actual parsing and formatting live in the vendored `chrome-devtools-frontend` trace engine.

## Key files
- `src/trace-processing/parse.ts` — the only file in this module.

## Key types & functions
- `TraceResult` — `{ parsedTrace: DevTools.TraceEngine.TraceModel.ParsedTrace; insights: ...TraceInsightSets | null }`. The successful parse result.
- `TraceParseError` — `{ error: string }`. Returned (not thrown) on failure.
- `traceResultIsSuccess(x)` — type guard; checks `'parsedTrace' in x`.
- `parseRawTraceBuffer(buffer, metadata?)` — async; decodes the buffer, JSON-parses it, runs the vendored engine, returns `TraceResult | TraceParseError`.
- `getTraceSummary(result, deviceScope?)` — returns a Markdown summary string built by the vendored `PerformanceTraceFormatter`, appended with call-frame & network format descriptions.
- `getInsightOutput(result, insightSetId, insightName, deviceScope?)` — returns `{output}` or `{error}` for a single named insight, formatted by the vendored `PerformanceInsightFormatter`.
- `InsightName` — `keyof DevTools.TraceEngine.Insights.Types.InsightModels` (the set of valid insight names, defined by the vendored bundle).

## How it works
A **module-level singleton engine** is created once at import time (first-party line, vendored constructor):
```ts
const engine = DevTools.TraceEngine.TraceModel.Model.createWithAllHandlers();
```
`createWithAllHandlers()` registers every trace handler so all insight types are available.

`parseRawTraceBuffer`:
1. Calls `engine.resetProcessor()` so the reused singleton starts clean each time.
2. Guards against an undefined buffer and an empty decoded string.
3. `TextDecoder().decode(buffer)` → `JSON.parse(...)`. The trace JSON may be either a bare `Event[]` array or `{ traceEvents: Event[] }`; both shapes are handled.
4. `await engine.parse(events, {metadata})` — vendored parsing. `metadata` carries optional `cpuThrottling` / `networkThrottling` (the tool passes the page's actual throttling so insights reflect real conditions).
5. `engine.parsedTrace()` → the `ParsedTrace`; `parsedTrace.insights` (may be `null`).
6. All errors are caught, logged via `logger?.(...)`, and returned as `TraceParseError` — this function does **not** throw.

`getTraceSummary` builds an `AgentFocus` from the parsed trace (`DevTools.AgentFocus.fromParsedTrace`), constructs a `PerformanceTraceFormatter`, and returns `formatTraceSummary()` plus the static `callFrameDataFormatDescription` / `networkDataFormatDescription` strings (both vendored) so the consuming model knows how to read call trees and network requests.

`getInsightOutput` looks up an insight set by id, then the named insight on `insightSet.model`, then formats it with `PerformanceInsightFormatter`. It returns descriptive error strings (rather than throwing) for: no insights at all, unknown `insightSetId`, or unknown `insightName` — those strings are surfaced to the caller/model.

## Vendored APIs used (all under `DevTools.*`)
- `TraceEngine.TraceModel.Model.createWithAllHandlers()`, `.resetProcessor()`, `.parse()`, `.parsedTrace()`
- `TraceEngine.TraceModel.ParsedTrace`, `TraceEngine.Types.Events.Event`, `TraceEngine.Insights.Types.{TraceInsightSets, InsightModels}`
- `AgentFocus.fromParsedTrace(...)`
- `PerformanceTraceFormatter` (+ static `callFrameDataFormatDescription`, `networkDataFormatDescription`)
- `PerformanceInsightFormatter`
- `CrUXManager.DeviceScope` (type only, for the optional `deviceScope` arg)

## Relationships
- **Depends on:** vendored `chrome-devtools-frontend` trace engine + formatters (via [`src/third_party/index.ts`](./third-party-workers.md)); `src/logger.ts`.
- **Used by:** [performance tool](../02-tools/groups/performance.md) — `stopTracingAndAppendOutput` calls `parseRawTraceBuffer`; `Response.attachTraceSummary` / `attachTraceInsight` call `getTraceSummary` / `getInsightOutput`.

## Gotchas & non-obvious details
- **Shared mutable singleton.** `engine` is one instance reused across all traces; correctness depends on `resetProcessor()` at the top of every parse. Trace parsing is therefore not safe to run concurrently.
- **Errors are values, not exceptions** — every caller must check `traceResultIsSuccess` (the tool throws on the error branch itself).
- **CrUX enrichment is external.** `getTraceSummary` reads `cruxFieldData` off `parsedTrace.metadata`, but it is the *performance tool* (`populateCruxData`) that fetches and attaches it before formatting — not this file.
- The insight *content* (which insights exist, what they report) is entirely owned by the vendored bundle.

## Update triggers
- `chrome-devtools-frontend` version bump in `package.json` (changes `InsightModels`, formatter output, or engine API).
- The set of valid insight names changes.
