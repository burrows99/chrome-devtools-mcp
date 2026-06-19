# Issue Formatter

> **Layer:** Formatters · **Sources:** `src/formatters/IssueFormatter.ts`, `src/issue-descriptions.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Formats DevTools `AggregatedIssue`s (e.g. CSP, deprecation, mixed-content) into concise list rows (title + count) or detailed output (markdown description, learn-more links, affected resources). Resolves issue-description markdown bundled from the DevTools front end and maps DevTools node/request ids onto the MCP's own stable uids/reqids.

## Key files
- `IssueFormatter.ts` — the formatter plus two text-conversion free functions.
- `../src/issue-descriptions.ts` — loads/serves the bundled issue-description markdown cache.

## Key types & functions
- `IssueFormatter` — main class; plain constructor (data is already in memory, no async factory).
- `IssueFormatterOptions` — `{id, requestIdResolver?, elementIdResolver?}`; resolvers map DevTools ids → MCP stable ids.
- `isValid()` — true only when a title was resolved; `McpResponse` drops invalid issues.
- `IssueConcise` / `IssueDetailed` — internal JSON shapes (detailed extends concise, adding `description?`, `links?`, `affectedResources`). Both carry `type: 'issue'`.
- `AffectedResource` — `{uid?, data?, request?}`.
- `ISSUE_UTILS` (`issue-descriptions.ts`) — `{loadIssueDescriptions, getIssueDescription}`. `loadIssueDescriptions()` reads every `*.md` under `third_party/issue-descriptions` into an in-memory cache once; `getIssueDescription(fileName)` returns cached markdown or `null`.

## How it works
**Concise** (`toString()` → `convertIssueConciseToString`):
```
msgid=<id> [issue] <title> (count: <count>)
```
**Detailed** (`toStringDetailed()` → `convertIssueDetailedToString`): `ID:` then `Message: issue> …` whose body is the processed markdown description (or title fallback), an optional `Learn more:` link list, and an optional `### Affected resources` block.

**Title/description resolution** (`#getTitle`, `#getDescription`): `issue.getDescription().file` names the markdown file; `ISSUE_UTILS.getIssueDescription(file)` fetches raw markdown; `DevTools.MarkdownIssueDescription.substitutePlaceholders(...)` fills substitutions. Title is then lexed via `DevTools.Marked` and extracted with `findTitleFromMarkdownAst`. Any failure logs via `logger?.(...)` and yields `undefined` (→ invalid issue). The detailed path strips a leading `# ` heading so it does not clash with the response's markdown hierarchy.

**Affected resources** (`#getAffectedResources`): iterates `issue.getAllIssues()`, `structuredClone`s each `details()`, then:
- Resolves `violatingNodeId` / `nodeId` / `documentNodeId` → `uid` via `elementIdResolver`, deleting the raw id from `data`.
- Resolves `request.requestId` → `reqid` via `requestIdResolver` (deleting it from `data`); otherwise keeps `request.url`.
- Deletes `errorType` and `frameId` (redundant/irrelevant to the MCP client).
- Remaining `data` is emitted as untyped JSON because the DevTools front-end detail types are not reusable here.

In text, each resource renders as `uid=… reqid=…|url=… data=<json>` (only present fields).

## Relationships
- **Depends on:** DevTools `AggregatedIssue`, `MarkdownIssueDescription`, `Marked` via `src/third_party/index.js`; `ISSUE_UTILS` from `src/issue-descriptions.ts`; `src/logger.ts`; caller-supplied `requestIdResolver`/`elementIdResolver` (from `McpContext`/`McpPage` stable-id maps).
- **Used by:** [`McpResponse`](../03-context-state/) — issues are interleaved with console messages; list path constructs `new IssueFormatter({id})`, checks `isValid()`, and calls `toString()`/`toJSON()`; drill-down passes resolvers and calls `toStringDetailed()`/`toJSONDetailed()`. `ConsoleFormatter.groupConsecutive` passes `IssueFormatter`s through unchanged.

## Gotchas & non-obvious details
- `loadIssueDescriptions()` must be called once at startup; the cache is module-global and skips reload if already populated.
- `request` is typed `string | number`; the text converter picks the `reqid=` vs `url=` prefix by `typeof === 'number'`, so a resolved reqid must be numeric.
- An issue with no resolvable markdown file is silently dropped (`isValid()` false) — not surfaced as an error to the agent.

## Update triggers
- Concise/detailed text format or affected-resource rendering changes.
- New node-id fields beyond `violatingNodeId`/`nodeId`/`documentNodeId` need resolving, or the `errorType`/`frameId` deletion list changes.
- The issue-description cache path or load semantics change.
