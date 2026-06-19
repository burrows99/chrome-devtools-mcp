# ID, String & Keyboard Utilities

> **Layer:** Utilities & shared types · **Sources:** `src/utils/id.ts`, `src/utils/string.ts`, `src/utils/keyboard.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Three independent leaf helpers: the resource-UID scheme (`id.ts`), Unicode-aware snake_case conversion (`string.ts`), and key-combo parsing/validation for synthetic keyboard input (`keyboard.ts`).

## Key functions / types
- `createIdGenerator(): () => number` (`id.ts`) — returns a closure yielding `1, 2, 3, …`; wraps back to `0` when it reaches `Number.MAX_SAFE_INTEGER`. Each generator instance has its own counter.
- `stableIdSymbol: unique symbol` (`id.ts`) — the symbol key under which a stable numeric ID is stashed on a collected object.
- `WithSymbolId<T> = T & { [stableIdSymbol]?: number }` (`id.ts`) — generic that brands a type as carrying an optional stable ID.
- `toSnakeCase(text: string): string` (`string.ts`) — converts camelCase/PascalCase/acronyms/letter-number boundaries to `snake_case`; returns `''` for empty input.
- `parseKey(keyInput: string): [KeyInput, ...KeyInput[]]` (`keyboard.ts`) — parses a `+`-joined combo (e.g. `"Control+Shift+KeyA"`) into a tuple of **primary key first, then modifiers in original order**; validates every token against an allowlist.

## How it works
**id.ts (the UID scheme — load-bearing).** Collectors call `createIdGenerator()` once, then attach `obj[stableIdSymbol] = generator()` to each resource as it is discovered. The model is shown the small integer; later, given that integer, the collector finds the live object by matching `item[stableIdSymbol] === id`. This is the mechanism behind accessibility-snapshot element references (`uid`s) and the resource handles for pages, service workers, and heap-snapshot nodes. See `PageCollector` (`getIdForResource` returns `obj[stableIdSymbol] ?? -1`; `getResourceById` matches by symbol) and `HeapSnapshotFormatter`.

**string.ts.** `toSnakeCase` applies five ordered regex passes (Unicode property escapes with the `u` flag): (1) letter→number boundary, (2) acronym→Word boundary (`APIFlags` → `API_Flags`), (3) lower/number→Upper boundary, then `toLowerCase()`, (4) collapse non-alphanumeric runs to a single `_`, (5) trim leading/trailing `_`.

**keyboard.ts.** `parseKey` scans char by char accumulating into `key`; a `+` flushes the current token (the `&& key` guard lets a literal `+` be its own key, e.g. `Shift++`). It rejects empty results and duplicate keys, then returns `[last, ...rest]` — i.e. the **last** token is treated as the primary key and everything before it as modifiers. `throwIfInvalidKey` checks the `validKeys` Set (digits, letters in both cases, F1–F24, numpad, media keys, punctuation, named keys) and throws with the full valid list on a miss.

## Relationships
- **Depends on:** `keyboard.ts` → `KeyInput` from `src/third_party/index.js`. `id.ts` and `string.ts` have no imports.
- **Used by:**
  - `id.ts` → `src/PageCollector.ts`, `src/ServiceWorkerCollector.ts`, `src/HeapSnapshotManager.ts` (all three import `createIdGenerator`, `stableIdSymbol`, `WithSymbolId`); `src/formatters/HeapSnapshotFormatter.ts` reads `stableIdSymbol`.
  - `string.ts` → `src/telemetry/flagUtils.ts` (normalizes flag names; wraps result in `stripUnderscoreBeforeNumber`).
  - `keyboard.ts` → `src/tools/input.ts` (the `press_key` handler: destructures `[key, ...modifiers]`, holds modifiers down, presses the key, releases modifiers in reverse).

## Gotchas & non-obvious details
- `parseKey` puts the **primary key last** in the input string but **first** in the returned tuple — `"Control+Shift+A"` → `[A, Control, Shift]`. Don't assume input order equals tuple order.
- `toSnakeCase`'s acronym rule splits `APIFlags` into `api_flags` but `API` alone stays `api` — boundary detection needs a following lowercase letter. `flagUtils` post-processes with `stripUnderscoreBeforeNumber` because rule (1) inserts an underscore before digits that callers don't always want.
- `createIdGenerator` wraps at `MAX_SAFE_INTEGER` rather than throwing, so IDs are not guaranteed globally unique forever — fine for session-scoped resources, but not a durable key.
- `WithSymbolId` makes the ID **optional** (`?`); unattached objects return `-1` via `getIdForResource`.

## Update triggers
- The UID scheme changes (generator semantics, symbol name, or how collectors attach/resolve IDs).
- `validKeys` set or the `parseKey` tuple ordering changes.
- `toSnakeCase` regex rules change.
