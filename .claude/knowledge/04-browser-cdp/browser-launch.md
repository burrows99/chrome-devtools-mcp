# Browser acquisition (launch / connect)

> **Layer:** Browser & CDP integration · **Sources:** `src/browser.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Obtain a single Puppeteer `Browser` for the process — either by **launching** a managed Chrome subprocess or **connecting** to an already-running one — and cleanly tear it down on shutdown.

## Key files
- `src/browser.ts` — all launch/connect/close logic and the target filter.

## Key types & functions
- `ensureBrowserConnected(options)` — connect to an existing Chrome; caches into the `browser` singleton, sets `browserMode = 'connected'`.
- `ensureBrowserLaunched(options)` — launch managed Chrome; sets `browserMode = 'launched'`.
- `launch(options: McpLaunchOptions)` — the raw `puppeteer.launch` wrapper (profile dir, args, viewport).
- `closeBrowser()` — shutdown hook: `close()` if launched, `disconnect()` if connected.
- `detectDisplay()` — Linux-only `$DISPLAY` recovery for headful mode.
- `makeTargetFilter(enableExtensions)` — Puppeteer `targetFilter` excluding `chrome://`, `chrome-untrusted://`, and (unless enabled) `chrome-extension://`.
- `type Channel = 'stable' | 'canary' | 'beta' | 'dev'`.

## How it works
**Singleton state.** Module-level `browser` and `browserMode` hold the one active connection. `ensureBrowser*` short-circuit if `browser?.connected`. Both assign `browserMode` *before* `browser` so a concurrent `closeBrowser()` never sees `browser` set with `browserMode` undefined (which would wrongly `disconnect()` and orphan a launched Chrome).

**Connect modes** (`ensureBrowserConnected`, in priority order):
- `wsEndpoint` → `browserWSEndpoint` (+ optional `wsHeaders`).
- `browserURL` → `browserURL`.
- `channel`/`userDataDir` (the `--auto-connect` path). With a `userDataDir`, it sets `autoConnect = true` and reads `DevToolsActivePort` from that dir to build a `ws://127.0.0.1:<port><path>` endpoint (validates the port is 1–65535). Without a userDataDir it maps `channel` to a Puppeteer `ChromeReleaseChannel` (`stable → 'chrome'`, else `chrome-<channel>`).
- Else throws (one of browserURL/wsEndpoint/channel/userDataDir required).

Connect always passes `targetFilter`, `defaultViewport: null`, `handleDevToolsAsPage: true`, and `blocklist`/`allowlist`.

**Launch** (`launch`): default (non-isolated) profile lives under `~/.cache/chrome-devtools-mcp[-cli]/chrome-profile[-<channel>]`; `--isolated` or an explicit `userDataDir` overrides. Always launches with `pipe: true` and `--hide-crash-restore-bubble`. Notable args: headless adds `--screen-info={3840x2160}`; `devtools` adds `--auto-open-devtools-for-tabs`. If no `executablePath`, the channel maps to a Puppeteer channel. After launch, optional `logFile` pipes the Chrome process stdout/stderr; optional `viewport` resizes the first page.

```ts
// channel → Puppeteer ChromeReleaseChannel (used in both launch & auto-connect)
channel === 'stable' ? 'chrome' : `chrome-${channel}`
```

## Relationships
- **Depends on:** `puppeteer-core` (re-exported via `src/third_party/index.ts`); Node `child_process`/`fs`/`os`.
- **Used by:** `src/index.ts:109` (`ensureBrowserConnected`/`ensureBrowserLaunched`); `src/bin/chrome-devtools-mcp-main.ts:55` (`closeBrowser` on stdin EOF / SIGTERM / SIGINT). CLI flags defined in `src/bin/chrome-devtools-mcp-cli-options.ts`.

## Gotchas & non-obvious details
- The `DevToolsActivePort` parsing is described as a TODO to re-expose via Puppeteer; it depends on Chrome 144+ remote-debugging being enabled in the target profile.
- Launch surfaces a friendlier error when Chrome is already running for a non-isolated `userDataDir` ("Use --isolated to run multiple browser instances").
- `detectDisplay()` is a no-op on win32/darwin; on Linux it greps other processes' `/proc/<pid>/environ` for a `DISPLAY` to recover one when launching headful from a context that lacks it.
- `--auto-connect` intentionally only passes `channel` (see `src/index.ts:116`); a bare `browserUrl`/`wsEndpoint` connect must not carry a channel.

## Update triggers
- A new connection mode or CLI flag is added.
- The default profile directory layout changes.
- Default launch args (`--hide-crash-restore-bubble`, `--screen-info`, `pipe`) change.
