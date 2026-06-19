# Emulation tools

> **Layer:** Tool system · **Sources:** `src/tools/emulation.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
A single `emulate` tool that configures device/network/locale emulation on the selected page: network throttling, CPU slowdown, geolocation, user agent, color scheme, viewport, and extra HTTP headers. All overrides are applied through `context.emulate`, which persists them so they can be restored after operations like Lighthouse audits.

## Key files
- `src/tools/emulation.ts` — defines the `emulate` page tool plus the `headerStringTransform` helper.

## Tools exposed
- `emulate` — page-scoped. `ToolCategory.EMULATION` (on by default), `readOnlyHint: false`, `blockedByDialog: true`. Params (all optional):
  - `networkConditions` — enum of `'Offline'` + keys of Puppeteer's `PredefinedNetworkConditions` (e.g. `Slow 3G`, `Fast 3G`). Omit to disable throttling.
  - `cpuThrottlingRate` — number `1`–`20` slowdown factor. Omit or `1` disables.
  - `geolocation` — string `"<lat>,<lng>"`, run through `geolocationTransform` -> `{latitude, longitude}`. Omit clears the override.
  - `userAgent` — string; empty string clears the override.
  - `colorScheme` — enum `'dark' | 'light' | 'auto'` (`auto` resets).
  - `viewport` — string `"<w>x<h>x<dpr>[,mobile][,touch][,landscape]"`, run through `viewportTransform` -> Puppeteer `Viewport`.
  - `extraHttpHeaders` — JSON-object string parsed by `headerStringTransform`; empty string -> `{}` (clears all). Headers persist across navigations until cleared.

## How it works
- The handler is a one-liner: `await context.emulate(request.params, page.pptrPage)` then `response.appendResponseLine('Emulation configured successfully')`. All logic lives in `context.emulate`.
- `geolocation` and `viewport` params are plain strings on the wire; Zod `.transform()` converts them to structured objects (`geolocationTransform` / `viewportTransform`, both in `src/tools/ToolDefinition.ts`) before reaching the handler.
- `headerStringTransform` `JSON.parse`s the headers string, rejecting non-object / array / null with a thrown error, and treats `''` as "clear" (`{}`).
- `throttlingOptions` is computed at module load as `['Offline', ...Object.keys(PredefinedNetworkConditions)]`, so the available throttle presets track Puppeteer's list.

## Relationships
- **Depends on:** `context.emulate` (and `context.restoreEmulation`) on the `Context` interface in `src/tools/ToolDefinition.ts`; Puppeteer's `PredefinedNetworkConditions` and `Viewport`; the shared `geolocationTransform`/`viewportTransform` transforms.
- **Used by:** the MCP `ToolHandler`. `lighthouse_audit` calls `context.restoreEmulation(page)` in its `finally` to undo emulation set here; `ContextPage` exposes `cpuThrottlingRate` and `networkConditions`, which the performance trace parser reads as throttling metadata.

## Gotchas & non-obvious details
- This is a single combined tool, not one tool per feature; every call applies the full param set (unspecified fields are typically left/cleared by `context.emulate`).
- Network and viewport are encoded as compact strings (not nested objects) and decoded by transforms — the wire schema for these is `string`, not the object shape.
- `extraHttpHeaders` rejects arrays and primitives; only a JSON object (or empty string to clear) is valid.
- Emulation set via this tool is what `context.restoreEmulation` reinstates after audits/operations that temporarily change it.

## Update triggers
- A param is added/removed or a transform changes in `src/tools/emulation.ts` (or in `viewportTransform`/`geolocationTransform`).
- Puppeteer's `PredefinedNetworkConditions` set changes (affects the `networkConditions` enum).
- `context.emulate` / `restoreEmulation` signatures change.
