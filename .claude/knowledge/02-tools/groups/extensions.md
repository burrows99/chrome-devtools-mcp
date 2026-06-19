# Extension tools

> **Layer:** Tool system · **Sources:** `src/tools/extensions.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Manage Chrome extensions inside the controlled browser: install an unpacked extension, uninstall, reload, list, and trigger an extension's default action. The whole category is experimental and off by default.

## Key files
- `src/tools/extensions.ts` — defines the five `*_extension(s)` tools (all plain `defineTool`, none page-scoped).

## Tools exposed
All use `ToolCategory.EXTENSIONS`, which is in `OFF_BY_DEFAULT_CATEGORIES` — disabled unless the category flag is enabled. None are `blockedByDialog`.
- `install_extension` — Params: `path` (string, absolute path to unpacked extension folder; `verifyFilesSchema: ['path']`). Calls `context.installExtension(path)`, responds `Extension installed. Id: <id>`. `readOnlyHint: false`.
- `uninstall_extension` — Params: `id` (string). Calls `context.uninstallExtension(id)`. `readOnlyHint: false`.
- `list_extensions` — No params. Calls `response.setListExtensions()` (name, ID, version, enabled status rendered by the response). `readOnlyHint: true`.
- `reload_extension` — Params: `id` (string). Looks up `context.getExtension(id)`, throws `Extension with ID ${id} not found.` if absent, then reinstalls from `extension.path`. `readOnlyHint: false`.
- `trigger_extension_action` — Params: `id` (string). Calls `context.triggerExtensionAction(id)`. `readOnlyHint: false`.

## How it works
- All handlers are thin wrappers over `Context` methods: `installExtension`, `uninstallExtension`, `getExtension`, `triggerExtensionAction`, and the `listExtensions` rendering path (`response.setListExtensions()`, which itself reads `context.listExtensions()`).
- `reload_extension` is implemented as "look up the extension, then `installExtension(extension.path)` again" — there is no dedicated reload API; reinstalling the same path is the reload.
- `install_extension` returns the new extension id in the response text, which is the id callers pass to the other four tools.

## Relationships
- **Depends on:** `Context.installExtension / uninstallExtension / getExtension / triggerExtensionAction / listExtensions` (interface in `src/tools/ToolDefinition.ts`); the `Extension` type from Puppeteer; `response.setListExtensions()`.
- **Used by:** the MCP `ToolHandler`. Category gating logic lives in `src/ToolHandler.ts` (`getCategoryStatus` + `OFF_BY_DEFAULT_CATEGORIES` in `src/tools/categories.ts`).

## Gotchas & non-obvious details
- Off by default — the whole category is in `OFF_BY_DEFAULT_CATEGORIES`; tools won't register/run without the extensions category flag enabled.
- `reload_extension` only works for unpacked extensions (it reinstalls from the on-disk `path`).
- `list_extensions` produces no inline text in the handler; output comes entirely from `response.setListExtensions()` rendering.
- Only `install_extension` validates a file path (`verifyFilesSchema: ['path']`); the id-based tools do no path checks.

## Update triggers
- A new `*_extension` tool is added or a param changes in `src/tools/extensions.ts`.
- The extensions category leaves/enters `OFF_BY_DEFAULT_CATEGORIES`.
- `Context` extension-management method names change.
