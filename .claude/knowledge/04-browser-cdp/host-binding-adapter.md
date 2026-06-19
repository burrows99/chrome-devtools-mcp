# Host binding adapter (InspectorFrontendHostAPI shim)

> **Layer:** Browser & CDP integration · **Sources:** `src/devtools/McpHostBindingAdapter.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The vendored `chrome-devtools-frontend` SDK normally runs inside a real DevTools window and talks to its embedder through `InspectorFrontendHostAPI` (file system, clipboard, metrics, zoom, AIDA, resource loading, etc.). Running the SDK headless in Node has no such host, so this file provides a **stub host** that satisfies the full interface — mostly no-ops — and implements only the handful of methods the MCP server actually needs.

## Key files
- `src/devtools/McpHostBindingAdapter.ts` — `BaseMcpHostBindingAdapter` (full no-op/throw base) and `McpHostBindingAdapter` (the real overrides).

## Key types & functions
- `BaseMcpHostBindingAdapter implements DevTools.Host.InspectorFrontendHostAPI.InspectorFrontendHostAPI` — implements *every* host method as a no-op, a `throw new Error('Not implemented')`, or (for callback-style methods) an immediate error callback (`doAidaConversation`, `registerAidaClientEvent`, `aidaCodeComplete`, `dispatchHttpRequest` all call back `{error: 'Not implemented'}`).
- `McpHostBindingAdapter extends BaseMcpHostBindingAdapter` — overrides only what's required, taking a `loadResource: (path) => Promise<string>` callback in its constructor.

## How it works
The base class exists to keep the surface honest: the comment states the subclass "should only implement methods that it needs to support," so anything not overridden is intentionally inert. The base `declare events` field stands in for the `EventTarget` the interface expects.

`McpHostBindingAdapter` overrides:
- `isolatedFileSystem()` → returns `null` (base would throw) — signals "no file system."
- `platform()` → maps `process.platform` to DevTools' `'mac' | 'windows' | 'linux'`.
- `zoomFactor()` → `1`.
- `isHostedMode()` → `true` (tells the SDK it's embedded, not standalone).
- `initialTargetId()` → resolves `null` (targets are created explicitly by the universe manager).
- `loadNetworkResource(urlString, headers, streamId, callback)` → validates the URL (`URL.canParse`), then delegates to the injected `loadResource`; on success it pumps content via `DevTools.Host.ResourceLoader.streamWrite(streamId, content)` and calls back `{statusCode: 200}`, otherwise `{statusCode: 404}`.

```ts
override isHostedMode(): boolean { return true; }
override platform(): string {
  switch (process.platform) { case 'darwin': return 'mac'; case 'win32': return 'windows'; default: return 'linux'; }
}
```

The single instance is installed globally via `DevTools.Host.InspectorFrontendHost.installInspectorFrontendHost(...)` in [DevtoolsUtils.ts](devtools-utils.md) `overrideDevToolsGlobals`, with `loadResource` wired to `McpContext.loadResource` (which fetches http(s) URLs or reads validated `file:` paths).

## Relationships
- **Depends on:** the vendored **`chrome-devtools-frontend`** `DevTools.Host.InspectorFrontendHostAPI` interface and `DevTools.Host.ResourceLoader` (via `src/third_party/index.ts`).
- **Used by:** [DevtoolsUtils.ts](devtools-utils.md) `overrideDevToolsGlobals` installs it; the `loadResource` callback originates in [McpContext](../03-context-state/mcp-context.md) (`McpContext.loadResource`, `src/McpContext.ts:955`).

## Gotchas & non-obvious details
- **Major vendored-frontend hub / version-sensitive:** `InspectorFrontendHostAPI` is a large, evolving interface. When `chrome-devtools-frontend@1.0.1646286` is bumped, new required methods may appear — they must be added to `BaseMcpHostBindingAdapter` or TypeScript will fail to compile against the interface.
- Methods that *throw* in the base (`isolatedFileSystem`, `platform`, `zoomFactor`, `isHostedMode`, `initialTargetId`) are exactly the ones the subclass is expected to override; a throw at runtime indicates an unsupported code path was reached.
- `loadNetworkResource` is the one genuinely functional host call — it's how the DevTools SDK pulls source maps / network resources, routed through MCP's own `loadResource` (which TODO-notes it does not yet honor the allow/block list).

## Update triggers
- `chrome-devtools-frontend` dependency is bumped (interface may gain/rename methods).
- The SDK starts requiring a host capability the server currently stubs (e.g. real AIDA, file system, or metrics).
