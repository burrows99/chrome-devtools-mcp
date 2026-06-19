# Maintaining this knowledge library

> How to keep `.claude/knowledge/` accurate as the codebase grows. This is the
> "increment or update as our codebase grows" contract. Treat it as a living
> spec — an agent or human should be able to follow it mechanically.

## Core principle
This library is a **derived artifact**. The code is the source of truth. Every doc
ends with an **`## Update triggers`** section listing the concrete code changes
that make it stale. When you make such a change, update the doc in the same PR.

## The doc template (use for every new file)
```
# <Title>

> **Layer:** <layer name> · **Sources:** `<comma-sep primary paths>` · **Verified at:** commit <sha> (<date>)

## Purpose
1–3 sentences.

## Key files
- `relative/path.ts` — one-line role

## Key types & functions
- `Name` — what it is/does (cite the file)

## How it works
Concise prose + bullets, real symbol names, tiny snippets (≤8 lines) only when they clarify.

## Relationships
- **Depends on:** <relative .md links>
- **Used by:** <relative .md links>

## Gotchas & non-obvious details
- ...

## Update triggers
- Concrete signals this doc is stale.
```

### Rules
- Keep each file focused, **≤120 lines**. Split rather than bloat — deep nesting is fine.
- **Relative** Markdown links between knowledge files; backticked `paths` for source.
- Be honest. Mark inferences as inferences; mark vendored code as vendored. Never
  invent an API — open the file.
- Update the **Verified at** stamp when you re-confirm a doc against the code.

## When the code changes — what to update
| Change | Update |
|--------|--------|
| New file under `src/tools/` | a new `02-tools/groups/<name>.md` + link it in `02-tools/README.md` |
| New field on `ToolDefinition` | [tool-definition.md](./02-tools/tool-definition.md) |
| New data collector / method on `McpContext` | [mcp-context.md](./03-context-state/mcp-context.md), [collectors.md](./03-context-state/collectors.md) |
| New formatter | a new `05-formatters/<name>.md` + `05-formatters/README.md` |
| New connection mode / `chrome-devtools-frontend` bump | [04-browser-cdp](./04-browser-cdp/README.md), [lighthouse-vendoring.md](./09-build-tooling/lighthouse-vendoring.md) |
| New metric/telemetry event | [07-telemetry](./07-telemetry/README.md) |
| New npm script / CI workflow | [scripts-reference.md](./09-build-tooling/scripts-reference.md), [ci-and-release.md](./09-build-tooling/ci-and-release.md) |
| New `src/` subsystem directory | a new layer (or sub-section) + [repository-map.md](./00-overview/repository-map.md) + the index table in [README.md](./README.md) |
| New shipped skill under `skills/` | [agent-skills.md](./11-skills-docs/agent-skills.md) |
| Convention/lint rule change | [12-conventions](./12-conventions/README.md) |

## Recommended workflow
### Incremental (default)
1. Make the code change.
2. Find the affected doc(s) via the table above or by searching the doc's
   **Sources** line for the file you touched:
   `grep -rl "src/path/you/changed" .claude/knowledge`
3. Update the doc, bump its **Verified at** stamp.
4. Run the link check (below) before committing.

### Full regeneration / large refactors
Re-run the fan-out that built this library: one subagent per layer, each given
the doc template above and its slice of `src/`. The [README.md](./README.md)
"Provenance" note describes the approach. Prefer incremental updates for routine
changes — full regen is for big structural shifts.

> Note: there is no `run-skill-generator` skill installed in this environment, so
> this guide *is* the maintenance procedure. If a skill is later added to automate
> it, point it at this template and the Update-triggers tables.

## Link-integrity check
Run from `.claude/knowledge/` after any edit. Catches broken relative links:
```bash
python3 - <<'PY'
import re, pathlib
root = pathlib.Path('.')
link_re = re.compile(r'\[[^\]]+\]\((\.{1,2}/[^)\s#]+)(?:#[^)]*)?\)')
broken = []
for md in root.rglob('*.md'):
    for m in link_re.finditer(md.read_text(encoding='utf-8')):
        if not (md.parent / m.group(1)).resolve().exists():
            broken.append((str(md), m.group(1)))
print("OK" if not broken else f"{len(broken)} broken:")
for f, t in broken: print(f"  {f} -> {t}")
PY
```

## Keeping it from rotting
- Stamp docs with the commit you verified against; stale stamps flag re-verification.
- If a doc and the code disagree, **trust the code** and fix the doc — don't
  paper over it.
- It's better to delete a doc than to leave it lying. Remove docs for deleted code.
