# Performance tools

> **Layer:** Tool system · **Sources:** `src/tools/performance.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Records and analyzes Chrome performance traces to surface frontend performance issues, Core Web Vitals (LCP, INP, CLS), and page-load bottlenecks. Trace recording is driven through Puppeteer's `tracing` API; parsing and insight formatting are delegated to the DevTools trace engine.

## Key files
- `src/tools/performance.ts` — defines the three `performance_*` tools and the shared `stopTracingAndAppendOutput` / `populateCruxData` helpers.

## Tools exposed
- `performance_start_trace` — page-scoped. Params: `reload` (boolean, default `true`), `autoStop` (boolean, default `true`), `filePath` (optional string; absolute or cwd-relative, `.json` or `.json.gz`). Starts a trace on the selected page, optionally navigating to `about:blank` then back to clear state, and (if `autoStop`) waits 5s, stops, and attaches the trace summary. `readOnlyHint: false`, `blockedByDialog: true`.
- `performance_stop_trace` — page-scoped. Params: `filePath` (optional, same as above). Stops the active trace and attaches summary; no-op if no trace is running. `readOnlyHint: false`, `blockedByDialog: true`.
- `performance_analyze_insight` — page-scoped. Params: `insightSetId` (string), `insightName` (string, e.g. `"DocumentLatency"`, `"LCPBreakdown"`). Attaches detailed output for one insight from the most recent recording. `readOnlyHint: true`, `blockedByDialog: false`.

All three use `ToolCategory.PERFORMANCE` (on by default).

## How it works
- `startTrace` guards on `context.isRunningPerformanceTrace()` (only one trace at a time) and sets the flag via `context.setIsRunningPerformanceTrace(true)`.
- Tracing categories are a fixed list kept in sync with the DevTools Timeline panel and Lighthouse (`-*`, `devtools.timeline`, `disabled-by-default-devtools.timeline*`, `v8.cpu_profiler`, etc.). Recording is `page.pptrPage.tracing.start({categories})`.
- `stopTracingAndAppendOutput` stops tracing, optionally gzips and saves the raw buffer via `context.saveFile`, then calls `parseRawTraceBuffer(buffer, {cpuThrottling, networkThrottling})`. On success it calls `context.storeTraceRecording(result)` and `response.attachTraceSummary(result)`; on failure it throws. The `finally` always resets the running flag.
- If `context.isCruxEnabled()`, `populateCruxData` fetches Chrome UX Report field data (public API key inline) and stuffs it into `result.parsedTrace.metadata.cruxFieldData` so the DevTools `PerformanceTraceFormatter` can include real-user data.
- `analyzeInsight` reads `context.recordedTraces().at(-1)` and calls `response.attachTraceInsight(trace, insightSetId, insightName)`; errors gracefully if no trace exists.

## Relationships
- **Depends on:** trace engine in [`src/trace-processing/parse.ts`](../../../../src/trace-processing/parse.ts) (`parseRawTraceBuffer`, `traceResultIsSuccess`, and the `TraceResult` / `InsightName` types used by the response). `attachTraceSummary` / `attachTraceInsight` on the response object render via `getTraceSummary` / `getInsightOutput` (DevTools `PerformanceTraceFormatter` / `PerformanceInsightFormatter`).
- **Used by:** the MCP `ToolHandler`; `lighthouse_audit` references `startTrace.name` in its description (it does NOT perform performance audits — directs users to this tool instead).

## Gotchas & non-obvious details
- `reload` and `autoStop` default to `true`. With them on, the page is reloaded; navigate to the target URL with `navigate_page` BEFORE starting the trace (called out in the param description).
- The 5-second wait in `autoStop` mode is hard-coded; the trace covers only that fixed window after load.
- Both start/stop are `blockedByDialog: true`, so an open dialog blocks them.
- The CrUX endpoint uses a deliberately public API key; CrUX fetching is gated behind `context.isCruxEnabled()` (the `--experimentalCrux`-style flag).
- The category list intentionally mirrors Chromium/Lighthouse sources; it must be kept in sync.

## Update triggers
- A new `performance_*` tool is added/renamed in `src/tools/performance.ts`.
- The trace category list or the `autoStop` 5s window changes.
- The `parseRawTraceBuffer` signature or CrUX population logic changes.
