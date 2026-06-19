# Lighthouse tool

> **Layer:** Tool system · **Sources:** `src/tools/lighthouse.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
A single `lighthouse_audit` tool that runs Lighthouse against the selected page for accessibility, SEO, best-practices, and agentic-browsing categories (explicitly NOT performance), saves JSON+HTML reports, and attaches a score/audit summary.

## Key files
- `src/tools/lighthouse.ts` — defines the `lighthouse_audit` page tool.

## Tools exposed
- `lighthouse_audit` — page-scoped. `ToolCategory.DEBUGGING` (note: NOT a `lighthouse` category — it lives under Debugging, on by default), `readOnlyHint: false`, `blockedByDialog: true`. Params:
  - `mode` — enum `'navigation' | 'snapshot'`, default `'navigation'`. `navigation` reloads & audits; `snapshot` audits the current state.
  - `device` — enum `'desktop' | 'mobile'`, default `'desktop'`. Selects the screen-emulation profile.
  - `outputDirPath` — optional string; directory for reports (`verifyFilesSchema: ['outputDirPath']`). If omitted, reports go to temporary files.

  Response: `response.attachLighthouseResult(output)` where `output` matches the `LighthouseData` shape — `summary` (`mode`, `device`, `url`, per-category `scores`, `audits.{failed,passed}`, `timing.total`) and `reports` (saved file paths).

## How it works
- Categories are hard-coded: `['accessibility', 'seo', 'best-practices', 'agentic-browsing']`; output formats are `['json', 'html']`.
- Builds a Lighthouse `Flags` object with `onlyCategories`, `output`, and `maxWaitForLoad: 30_000`. Device picks one of two `screenEmulation` profiles: desktop `1350x940@1`, mobile `412x823@1.75`.
- Runs `navigation(page.pptrPage, url, {flags})` or `snapshot(page.pptrPage, {flags})` (imported Lighthouse runners). On no result -> throws `Lighthouse audit failed.`
- A `finally` always calls `context.restoreEmulation(page)` so Lighthouse's emulation does not leak into later tools.
- Reports are generated via `generateReport(lhr, format)` and saved with `context.saveFile(data, <dir>/report, '.<format>')` when `outputDirPath` is set, else `context.saveTemporaryFile`. Saves run via `Promise.allSettled`; any rejection re-throws.
- Summary is derived from the `lhr`: `scores` from `lhr.categories`, `failed` = audits with `score !== null && < 1`, `passed` = audits with `score === 1`.

## Relationships
- **Depends on:** Lighthouse `navigation` / `snapshot` / `generateReport` and the `Flags` / `RunnerResult` / `OutputMode` types (re-exported via `src/third_party/index.js`); `context.saveFile` / `saveTemporaryFile` / `restoreEmulation`; `response.attachLighthouseResult` and the `LighthouseData` interface in `src/tools/ToolDefinition.ts`.
- **Used by:** the MCP `ToolHandler`. The description references `startTrace.name` (`performance_start_trace`) to redirect users wanting performance audits to the [performance tools](./performance.md).

## Gotchas & non-obvious details
- Performance is deliberately excluded; the description points to `performance_start_trace` instead.
- Despite the file/feature name, the tool's category is `DEBUGGING`, not a dedicated lighthouse category — so it is on by default and gated by the debugging flag.
- `lighthouse_audit` mutates emulation while running, but always restores it via `context.restoreEmulation` in `finally` (shared with the [emulation tool](./emulation.md)).
- `navigation` mode reloads the page; ensure the page is at the intended URL first.
- Report file naming is fixed to `report.json` / `report.html`; multiple runs into the same dir will collide unless the saver disambiguates.

## Update triggers
- The audited categories, output formats, device profiles, or `maxWaitForLoad` change in `src/tools/lighthouse.ts`.
- The Lighthouse version is bumped (runner/flag/`lhr` shape changes).
- `LighthouseData` / `attachLighthouseResult` shape changes.
