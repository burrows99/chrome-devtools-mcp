# Tool Group: Screencast

> **Layer:** Tool system · **Sources:** `src/tools/screencast.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Two tools to record a video (screencast) of the selected page and stop it. Both are category `DEBUGGING` and gated behind the experimental condition `experimentalScreencast`. Recording requires `ffmpeg`.

## Key files
- `src/tools/screencast.ts` — `screencast_start`, `screencast_stop`, `generateTempFilePath` helper.

## Key types & functions
Both are `definePageTool`, category `DEBUGGING`, `readOnlyHint: false`, `blockedByDialog: false`, condition `['experimentalScreencast']`.

- `startScreencast` (MCP name `screencast_start`, factory form) — schema `{filePath?}`; `verifyFilesSchema: ['filePath']`. Supported extensions `.webm`, `.mp4`.
- `stopScreencast` (MCP name `screencast_stop`) — empty schema `{}`.

## How it works
- **Single active recording:** `screencast_start` first checks `context.getScreenRecorder()`; if non-null it returns an error line (only one recording at a time). `screencast_stop` errors if there is none.
- **Path/format resolution:** if `filePath` omitted, `generateTempFilePath()` makes an `mkdtemp` dir with `screencast.mp4`. The requested extension is matched case-insensitively against `.webm`/`.mp4`; an explicit unsupported extension throws; a missing extension falls back to `.mp4`. The matched extension drives `VideoFormat`. The path is resolved/normalized via `ensureExtension` (`src/utils/files.ts`).
- **Start:** `page.pptrPage.screencast({path, format, ffmpegPath: args?.experimentalFfmpegPath})`. The recorder + path are stored with `context.setScreenRecorder({recorder, filePath})`. On failure, if the dir was auto-generated it's removed; a missing-ffmpeg `ENOENT` is rethrown as a friendly "ffmpeg is required..." message.
- **Stop:** `data.recorder.stop()` then `context.setScreenRecorder(null)` in `finally`, reporting the saved path.

## Relationships
- **Depends on:** [ContextPage](../tool-definition.md) (`pptrPage.screencast`), [McpContext](../../03-context-state/mcp-context.md) (`getScreenRecorder`/`setScreenRecorder`), `ensureExtension` (`src/utils/files.ts`), Puppeteer `ScreenRecorder`/`VideoFormat`.
- **Used by:** agents capturing interaction recordings; experimental.

## Gotchas & non-obvious details
- Recorder state lives on the `McpContext`, so start/stop must target the same context; only one recording can be in flight server-wide.
- `--experimental-ffmpeg-path` lets users point at a custom ffmpeg binary; without ffmpeg, start fails with an explicit install hint.
- An explicitly requested unsupported extension is rejected rather than silently rewritten to `.mp4`, to avoid changing both format and output path.

## Update triggers
- A tool is added/removed in `src/tools/screencast.ts`.
- Supported video extensions/formats or the ffmpeg-path wiring change.
