# Collectors (network, console, service-worker)

> **Layer:** Context & state · **Sources:** `src/PageCollector.ts`, `src/ServiceWorkerCollector.ts`, `src/utils/id.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Collectors capture per-page event streams (network requests, console messages, page errors, DevTools aggregated issues) and per-extension service-worker logs, keeping them keyed by page and split by navigation. Each captured item is tagged with a stable numeric ID so it can be referenced across tool calls.

## Key files
- `src/PageCollector.ts` — generic `PageCollector<T>` + `NetworkCollector`, `ConsoleCollector`, and the `UncaughtError` class / `PageEventSubscriber`.
- `src/ServiceWorkerCollector.ts` — `ServiceWorkerConsoleCollector` + `ServiceWorkerSubscriber`.
- `src/utils/id.ts` — `createIdGenerator`, `stableIdSymbol`, `WithSymbolId<T>`.

## Key types & functions
- `PageCollector<T>` — base: `WeakMap<Page, Array<Array<WithSymbolId<T>>>>` storage (`src/PageCollector.ts:50`).
- `NetworkCollector extends PageCollector<HTTPRequest>` — listens to `request`; custom navigation split (`:361`).
- `ConsoleCollector extends PageCollector<ConsoleMessage | Error | AggregatedIssue | UncaughtError>` — adds a `PageEventSubscriber` per page (`:223`).
- `UncaughtError` — wraps `Protocol.Runtime.ExceptionDetails` + a `targetId` (`:31`).
- `ServiceWorkerConsoleCollector` — `Map<extensionId, logs[]>`, ring-buffered (`src/ServiceWorkerCollector.ts:69`).

## How PageCollector works
- **Storage model.** Per page, an array of navigations, each a list of collected items. Newest navigation is index `0`. `maxNavigationSaved = 3`.
- **Init / lifecycle.** `init(pages)` adds each page and subscribes to browser `targetcreated`/`targetdestroyed`; new page targets are auto-added, destroyed ones cleaned up (listeners removed, storage entry dropped).
- **Stable IDs.** A per-page `createIdGenerator()` stamps every item with `item[stableIdSymbol] = n` as it's collected. `getIdForResource`, `getById`, `find` use this symbol. IDs are unique within a page across navigations.
- **Navigation split.** A `framenavigated` listener on the main frame calls `splitAfterNavigation`, which `unshift`es a new empty navigation list and truncates to `maxNavigationSaved`.
- **Reads.** `getData(page, includePreservedData?)` returns just the current navigation (`navigations[0]`) by default, or all retained navigations (oldest→newest) when preserved data is requested.

## NetworkCollector specifics
Overrides `splitAfterNavigation`: instead of starting empty, it finds the last main-frame navigation request and keeps everything from that request onward in the new current navigation (so in-flight/just-started requests aren't orphaned across the split). Default listener collects every `request`; the test variant filters out `favicon.ico`.

## ConsoleCollector & PageEventSubscriber
`ConsoleCollector` adds, per page, a `PageEventSubscriber` that bridges several sources into the page's event stream:
- **Console messages** & **page errors** — via the base listeners passed from `McpContext` (`console`, `uncaughtError`, `devtoolsAggregatedIssue`).
- **Uncaught exceptions** — a CDP `Runtime.exceptionThrown` listener emits an `UncaughtError(details, targetId)` as the `uncaughtError` page event.
- **DevTools issues** — raw `issue` events are mapped through DevTools' `createIssuesFromProtocolIssue` into an `IssueAggregator`; deduped aggregated issues are re-emitted as `devtoolsAggregatedIssue`. On main-frame navigation the aggregator and dedupe sets (`#seenKeys`, `#seenIssues`) are reset.
The collected union type is `ConsoleMessage | Error | AggregatedIssue | UncaughtError`; `McpResponse` discriminates them when formatting (`'args' in item`, `instanceof UncaughtError`, `instanceof AggregatedIssue`).

## ServiceWorkerConsoleCollector
Separate from the page collectors because extension service workers aren't pages. Keyed by `extensionId` (extracted from the `chrome-extension://<id>/…` origin). Per matching `service_worker` target it creates a `ServiceWorkerSubscriber` that opens its own CDP session, enables `Runtime`, and listens for `console` (worker) and `Runtime.exceptionThrown` (→ `UncaughtError`). Storage is a flat ring buffer per extension capped at `#maxLogs` (default 1000) — note: **no per-navigation split** here, unlike page collectors. Items get stable IDs from a single shared `#idGenerator`.

## Relationships
- **Depends on:** [Browser & CDP](../04-browser-cdp/README.md) (Puppeteer `Browser`/`Page`/`Target`/`CDPSession`, `FakeIssuesManager`, DevTools `IssueAggregator`), [Utilities](../08-utilities/README.md) (`createIdGenerator`, `stableIdSymbol`).
- **Used by:** [McpContext](mcp-context.md) (owns three collector instances), [McpResponse](mcp-response.md) (reads data when building network/console output), [Formatters](../05-formatters/README.md) (format the collected items).

## Gotchas & non-obvious details
- Storage is a `WeakMap<Page, …>` — entries vanish when the page is GC'd; collectors hold no strong page references.
- Stable IDs are stored on the item itself via a `Symbol` (`WithSymbolId<T>`), not in a side table — they survive as long as the item object does and recycle at `Number.MAX_SAFE_INTEGER`.
- Page collectors keep only the **last 3 navigations**; service-worker logs keep the **last 1000 messages**.
- `NetworkCollector` and `ConsoleCollector` differ in navigation-split semantics (network keeps the trailing navigation request; console just starts fresh and resets issue dedupe).
- Subscriber teardown swallows `Target closed`/`Session closed` errors (expected during shutdown).

## Update triggers
- New event source bridged into a collector, change to navigation-split or retention limits, new collected type, or a change to the stable-ID scheme.
