# Snapshot Formatter

> **Layer:** Formatters · **Sources:** `src/formatters/SnapshotFormatter.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Renders an accessibility (a11y) tree — a `TextSnapshot` of `TextSnapshotNode`s — into an indented, token-efficient text outline (the agent's primary view of the page DOM) or an equivalent nested JSON object. Each node line carries a stable `uid` that other tools use to target elements.

## Key files
- `SnapshotFormatter.ts` — the formatter, the boolean-property map, and the excluded-attribute set.

## Key types & functions
- `SnapshotFormatter` — plain constructor over a `TextSnapshot`; no async factory, no detailed variant.
- `toString()` — full indented text outline (depth × 2 spaces per level).
- `toJSON()` — nested object: each node's attribute map plus a `children` array when non-empty.
- `#getAttributes(node)` — builds the ordered text attribute list for one node.
- `#getAttributesMap` / `#extractedAttributes` — build the attribute key/value map used by JSON and (with `excludeSpecial`) by text.
- `booleanPropertyMap` — maps DevTools boolean props to capability names: `disabled→disableable`, `expanded→expandable`, `focused→focusable`, `selected→selectable`.
- `excludedAttributes` — `Set` of internal fields never emitted: `id`, `role`, `name`, `elementHandle`, `children`, `backendNodeId`, `loaderId`.

## How it works
There is **no concise/detailed split** here — a snapshot is already a whole-tree document; the two outputs are text vs JSON of the *same* data.

**Text** (`toString` → `#formatNode`, recursive): each node is
```
<indent>uid=<id> <role> "<name>" <attr…> [ [selected in the DevTools Elements panel]]
```
- `role === 'none'` renders as `ignored`.
- `name` is quoted.
- Remaining attributes are appended in **sorted key order**: a present `booleanPropertyMap` capability is emitted, a `true` value emits the bare attr name, and string/number values emit `attr="value"`.
- The selected node (matching `snapshot.selectedElementUid`) gets the ` [selected …]` marker.

**Selected-element note** — `toString` prepends a warning when the page has a selected element but the snapshot is non-verbose and does not include it, advising the agent to request a verbose snapshot.

**JSON** (`toJSON` → `#nodeToJSON`, recursive): `structuredClone` of the attribute map (special fields included: `id`, `role`, `name`), with `children` appended when present.

## Relationships
- **Depends on:** `TextSnapshot` / `TextSnapshotNode` from [`src/TextSnapshot.ts` and `src/types.ts`](../03-context-state/) — the a11y collection layer produces the tree; this formatter only renders it.
- **Used by:** [`McpResponse`](../03-context-state/) — builds `new SnapshotFormatter(textSnapshot)`. The text outline may be written to a snapshot file (`new TextEncoder().encode(formatter.toString())` → `snapshotFilePath`) or inline; the JSON goes to `structuredContent.snapshot`, optionally TOON-encoded.

## Gotchas & non-obvious details
- Attribute ordering is deterministic via `Object.keys(node).sort()` — output is stable across runs.
- `#getAttributesMap`/`#extractedAttributes` overlap intentionally ("re-implementing the exact logic … to be safe"); text and JSON share the same extraction rules so they cannot drift.
- Verbosity is a property of the upstream `TextSnapshot` (`snapshot.verbose`), not a formatter option — the formatter only reacts to it for the selected-element note.

## Update triggers
- Node line format, indentation, or the selected-element marker/note changes.
- `booleanPropertyMap` or `excludedAttributes` membership changes.
- The `TextSnapshotNode` shape gains fields that should (or should not) be emitted.
