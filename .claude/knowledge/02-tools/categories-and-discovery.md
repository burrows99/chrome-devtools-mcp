# Categories & Discovery

> **Layer:** Tool system · **Sources:** `src/tools/categories.ts`, `src/tools/tools.ts`, `src/ToolHandler.ts`, `src/index.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Explains how tools are grouped into categories, how a category (or experimental condition) gates a tool on/off, and how `createTools` collects every group module into the single sorted list the server registers.

## Key files
- `src/tools/categories.ts` — `ToolCategory` enum, `labels`, `OFF_BY_DEFAULT_CATEGORIES`.
- `src/tools/tools.ts` — `createTools(args)` aggregation + sort.
- `src/ToolHandler.ts` — `buildFlag`, gating logic (mirrored in [tool-handler](tool-handler.md)).

## Key types & functions
- `enum ToolCategory` — `INPUT='input'`, `NAVIGATION='navigation'`, `EMULATION='emulation'`, `PERFORMANCE='performance'`, `NETWORK='network'`, `DEBUGGING='debugging'`, `EXTENSIONS='extensions'`, `THIRD_PARTY='experimentalThirdParty'`, `MEMORY='memory'`, `WEBMCP='experimentalWebmcp'`.
- `labels` — human strings per category (used in disabled-tool messages and generated docs).
- `OFF_BY_DEFAULT_CATEGORIES` — `[EXTENSIONS, THIRD_PARTY, WEBMCP]`; off unless their flag is set.
- `createTools(args)` — returns `ToolDefinition[]`, alphabetically sorted by `name`.

## How it works
**Discovery / aggregation** (`createTools`): in non-slim mode it spreads `Object.values(...)` of every group module (`console`, `emulation`, `extensions`, `input`, `lighthouse`, `memory`, `network`, `pages`, `performance`, `screencast`, `screenshot`, `script`, `snapshot`, `thirdPartyDeveloper`, `webmcp`). In slim mode it uses only `slim/tools`. Each value that is a function (factory tool) is called with `args`; the rest are used as-is. The list is then `sort`ed by `name`.

**Category → flag gating** (`ToolHandler`): each category maps to a CLI flag via `buildFlag` — `network` → `categoryNetwork`, `experimentalThirdParty` → `categoryExperimentalThirdParty`, etc. A tool is disabled when:
- its category is in `OFF_BY_DEFAULT_CATEGORIES` and the flag is unset/false, **or**
- its category flag is explicitly `false`, **or**
- any `annotations.conditions` flag (e.g. `experimentalVision`, `experimentalScreencast`, `experimentalInteropTools`, `experimentalNavigationAllowlist`) is falsy.

Disabled tools produce a `buildDisabledMessage` telling the user which `chrome-devtools start <flag>=true` to run.

**Registration**: `src/index.ts` iterates the sorted list, builds a `ToolHandler`, and registers it unless `shouldRegister` is false.

## Relationships
- **Depends on:** group modules under `src/tools/`, [ToolHandler](tool-handler.md), CLI args (`src/bin/chrome-devtools-mcp-cli-options.ts`).
- **Used by:** [README](README.md) group index; [slim mode](slim-mode.md) is the alternate discovery path.

## Gotchas & non-obvious details
- Category enum *values* are the flag stems: experimental categories embed `experimental` in the value (`experimentalThirdParty`, `experimentalWebmcp`) so `buildFlag` yields `categoryExperimentalThirdParty`.
- A single group module can host tools of different categories — `pages.ts` exports `resize_page` (EMULATION) and `handle_dialog` (INPUT) alongside NAVIGATION tools; categorization is per-tool, not per-file.
- `conditions` are separate from categories: a tool can be in an enabled category yet still gated by an experimental condition flag (e.g. `click_at` needs `experimentalVision`).
- Sorting is purely cosmetic (stable tool ordering for clients/docs); it does not affect behavior.

## Update triggers
- A category is added/renamed in `ToolCategory` or `OFF_BY_DEFAULT_CATEGORIES`.
- A new group module is added to the `createTools` spread list.
- A new experimental `conditions` flag is introduced.
