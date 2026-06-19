# Tool Group: Screenshot

> **Layer:** Tool system · **Sources:** `src/tools/screenshot.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
A single tool, `take_screenshot`, that captures the page, the viewport, or a specific element as PNG/JPEG/WebP — inline, to disk, or auto-spilled to a temp file when large.

## Key files
- `src/tools/screenshot.ts` — `take_screenshot` plus `getSourceBox`/`computeDownscaleClip` helpers.

## Key types & functions
- `screenshot` (MCP name `take_screenshot`, `definePageTool` factory) — category **DEBUGGING**, `readOnlyHint: false` (filePath), `blockedByDialog: true`. `verifyFilesSchema: ['filePath']`.
- Schema: `{format(png|jpeg|webp, default from CLI or png), quality?(0-100, jpeg/webp only), uid?, fullPage?, filePath?}`.
- Factory reads CLI args `screenshotFormat`, `screenshotQuality`, `screenshotMaxWidth`, `screenshotMaxHeight` to set defaults and downscale bounds.

## How it works
- Rejects `uid` + `fullPage` together.
- Resolves an element handle when `uid` is given (`request.page.getElementByUid`), else captures the page.
- **Downscaling:** when `--screenshot-max-width`/`--screenshot-max-height` are set, `getSourceBox` computes the source box (element box, full-page scroll size, or viewport) and `computeDownscaleClip` derives a `ScreenshotClip` with a `scale` (smaller of width/height scale wins, preserves aspect ratio; skips no-op or sub-pixel results). With a clip, `page.screenshot({clip, ...})` lets CDP downscale (relies on Puppeteer's `captureBeyondViewport=true` default so below-the-fold captures work).
- Capture path: clip → `page.screenshot({clip})`; else element → `element.screenshot()`; else → `page.screenshot({fullPage})`. All use `optimizeForSpeed: true`; `quality` is undefined for PNG.
- **Output routing:** if `filePath` → `context.saveFile(screenshot, filePath, ext)`; else if bytes `>= 2_000_000` → `context.saveTemporaryFile(...)` and report the path; else → `response.attachImage({mimeType, data: base64})` inline.
- Extension is narrowed manually (`.png`/`.jpeg`/`.webp`) because the factory form widens the Zod literal union.

## Relationships
- **Depends on:** [ContextPage](../tool-definition.md) (`pptrPage`, `getElementByUid`), [McpContext](../../03-context-state/mcp-context.md) (`saveFile`, `saveTemporaryFile`), [McpResponse](../../03-context-state/mcp-response.md) (`attachImage`, `appendResponseLine`), Puppeteer `ScreenshotClip`/`BoundingBox`.
- **Used by:** agents needing visual context; `uid` ties to [snapshot](snapshot.md).

## Gotchas & non-obvious details
- Inline images are capped at ~2MB: anything `>= 2_000_000` bytes is written to a temp file and only the path returned, so large captures never bloat the MCP payload.
- `quality` is silently ignored for PNG.
- Default format comes from the CLI (`--screenshot-format`), defaulting to `png` when unset.
- The slim variant (`screenshot`) is a much simpler, param-less PNG-to-temp tool — see [slim mode](../slim-mode.md).

## Update triggers
- The schema, downscale logic, or output-routing thresholds change.
- A new screenshot CLI flag is wired into the factory.
