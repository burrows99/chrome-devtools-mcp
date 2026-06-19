# Design principles

> **Layer:** Conventions · **Sources:** `docs/design-principles.md`, `README.md` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
The product philosophy every tool and response must follow. These are the
"rough guidelines, applied with nuance" from `docs/design-principles.md` — but in
practice they explain *why* the code is shaped the way it is (formatters,
slim mode, file-reference outputs, self-healing errors).

## The seven principles
1. **Agent-Agnostic API** — build on standards (MCP); don't lock into one LLM.
   Interoperability first.
2. **Token-Optimized** — return semantic summaries, not raw dumps. "LCP was 3.2s"
   beats 50k lines of JSON. Large data belongs in files. → drives the
   [formatters](../05-formatters/README.md) and the concise/detailed split.
3. **Small, Deterministic Blocks** — composable tools (click, screenshot), not
   magic do-everything buttons. → shapes the [tool groups](../02-tools/README.md).
4. **Self-Healing Errors** — return actionable errors with context and likely
   fixes, so the agent can recover.
5. **Human-Agent Collaboration** — output must be machine-readable (structured)
   *and* human-readable (summaries). → [McpResponse](../03-context-state/mcp-response.md).
6. **Progressive Complexity** — simple high-level defaults, optional advanced
   args for power users.
7. **Reference over Value** — for heavy assets (screenshots, traces, videos),
   return a file path / resource URI, never the raw stream. (MCP clients with
   native asset handling may be an exception, e.g. inline images.)

## Where they show up in code
- Token-optimization & reference-over-value → [formatters](../05-formatters/README.md), [performance tools](../02-tools/groups/performance.md).
- Progressive complexity & slim defaults → [slim-mode](../02-tools/slim-mode.md).
- Self-healing errors → error handling in [ToolHandler](../02-tools/tool-handler.md).

## Update triggers
- `docs/design-principles.md` changes, or a new cross-cutting principle is adopted.
