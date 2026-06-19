# Tool Group: Input

> **Layer:** Tool system · **Sources:** `src/tools/input.ts`, `src/tools/pages.ts` (handle_dialog) · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Tools that simulate user input on the selected page: clicking, hovering, filling form fields, typing, dragging, pressing keys, uploading files, and answering JS dialogs. All resolve elements by `uid` from the latest a11y [snapshot](snapshot.md).

## Key files
- `src/tools/input.ts` — all input tools except `handle_dialog`.
- `src/tools/pages.ts` — `handle_dialog` (category INPUT, lives in pages module).

## Key types & functions
All are `definePageTool`, category `INPUT` (except where noted), `readOnlyHint: false`, and most are `blockedByDialog: true`.

- `click` — `{uid, dblClick?, includeSnapshot?}`. Resolves the element, and if the a11y node `role === 'option'` (and not dblClick) selects the native `<select>` option via `selectNativeSelectOption`; otherwise `handle.asLocator().click({count})`.
- `click_at` — `{x, y, dblClick?, includeSnapshot?}`; gated by condition `experimentalVision`. Uses `pptrPage.mouse.click(x, y)`.
- `hover` — `{uid, includeSnapshot?}`; `handle.asLocator().hover()`.
- `fill` — `{uid, value, includeSnapshot?}`. Delegates to `fillFormElement` (combobox→select option, checkbox/radio/switch→`"true"`/`"false"`, otherwise `.fill(value)` with a timeout that grows 10ms/char).
- `fill_form` — `{elements: Array<{uid, value}>, includeSnapshot?}`; loops `fillFormElement` over each. Description tells the model to prefer this over multiple `fill`/`click` calls.
- `type_text` — `{text, submitKey?}`; `keyboard.type(text)` then optional `keyboard.press(submitKey)`. Not uid-based — types into the already-focused element.
- `drag` — `{from_uid, to_uid, includeSnapshot?}`; `fromHandle.drag(toHandle)`, 50ms pause, `toHandle.drop(fromHandle)`.
- `upload_file` — `{uid, filePath, includeSnapshot?}`; `verifyFilesSchema: ['filePath']`. Tries `handle.uploadFile(filePath)`; on failure falls back to clicking and `waitForFileChooser` then `fileChooser.accept([filePath])`.
- `press_key` — `{key, includeSnapshot?}`; `parseKey` splits modifiers (Control/Shift/Alt/Meta) from the key, holds modifiers down, presses, releases in reverse.
- `handle_dialog` — `{action: 'accept'|'dismiss', promptText?}`; category INPUT, `blockedByDialog: false`. Calls `page.getDialog()`, accepts/dismisses, then `page.clearDialog()`.

## How it works
- Most handlers wrap the action in `page.waitForEventsAfterAction(...)` and `response.attachWaitForResult(result)` so the response reports navigations/network/console settling triggered by the input.
- `includeSnapshot?` (default false) calls `response.includeSnapshot()` to append a fresh a11y snapshot.
- Element resolution: `request.page.getElementByUid(uid)` returns a Puppeteer `ElementHandle`, always `dispose()`d in `finally`. `request.page.getAXNodeByUid(uid)` gives the snapshot node (for option/combobox logic).
- Errors during interaction route through `handleActionError(error, uid)`, which throws a uniform "did not become interactive within the configured timeout" message with the original as `cause`.

## Relationships
- **Depends on:** [ContextPage](../tool-definition.md) (`getElementByUid`, `getAXNodeByUid`, `waitForEventsAfterAction`, `getDialog`/`clearDialog`), [McpContext](../../03-context-state/mcp-context.md) (`fill`/`fill_form` cast `context` to `McpContext`), `parseKey` (`src/utils/keyboard.ts`), `WaitForHelper`.
- **Used by:** agents after a [snapshot](snapshot.md) provides uids.

## Gotchas & non-obvious details
- `fill`/`fill_form` reuse `fillFormElement`, which special-cases comboboxes-with-options (treated as `<select>`) and toggles (require literal `"true"`/`"false"`).
- `type_text` has no `uid` — it relies on a prior `click`/`fill` to focus the field.
- `upload_file` is the only input tool with `verifyFilesSchema`.
- `click`'s native-option path exists because option a11y nodes don't carry their `value`; it climbs to the parent `<select>` and fills the element's `value` property.

## Update triggers
- A tool is added/removed in `src/tools/input.ts`, or `handle_dialog` moves.
- The combobox/toggle/native-select logic in `fillFormElement` changes.
