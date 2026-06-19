# Knowledge Library — `chrome-devtools-mcp`

> A layered, navigable reference for this codebase. Built to be **incremented and
> updated as the code grows** — see [MAINTENANCE.md](./MAINTENANCE.md).
>
> **Verified at:** commit afa5622 (2026-06-19) · 84 docs across 13 layers

## What this is
A long-term textual knowledge base for the Chrome DevTools MCP server. It is
organized **top-down**: layer 00 is the 10,000-foot view; each higher number
descends closer to individual modules and files. Every doc follows a fixed
template (Purpose → Key files → Key types/functions → How it works →
Relationships → Gotchas → Update triggers) and cross-links to its neighbors.

**New here?** Read [00-overview](./00-overview/README.md) first, then dive into
whichever layer you're working in.

## Layers
| # | Layer | Index | Covers |
|---|-------|-------|--------|
| 00 | Overview | [↗](./00-overview/README.md) | what it is, [architecture](./00-overview/architecture.md), [repo map](./00-overview/repository-map.md), [stack](./00-overview/tech-stack.md), [glossary](./00-overview/glossary.md) |
| 01 | Entrypoints & process model | [↗](./01-entrypoints/README.md) | `src/bin/*`, [server bootstrap](./01-entrypoints/server-bootstrap.md), [CLI](./01-entrypoints/cli.md), [daemon](./01-entrypoints/daemon.md) |
| 02 | Tool system | [↗](./02-tools/README.md) | [ToolDefinition](./02-tools/tool-definition.md), [ToolHandler](./02-tools/tool-handler.md), [categories](./02-tools/categories-and-discovery.md), [slim mode](./02-tools/slim-mode.md), 15 [tool groups](./02-tools/README.md) |
| 03 | Context & state | [↗](./03-context-state/README.md) | [McpContext](./03-context-state/mcp-context.md) (the hub), [McpPage](./03-context-state/mcp-page.md), [McpResponse](./03-context-state/mcp-response.md), [collectors](./03-context-state/collectors.md), [snapshots](./03-context-state/text-snapshot.md), [concurrency](./03-context-state/concurrency.md) |
| 04 | Browser & CDP | [↗](./04-browser-cdp/README.md) | [launch/connect](./04-browser-cdp/browser-launch.md), [connection adapters](./04-browser-cdp/connection-adapters.md), [host bindings](./04-browser-cdp/host-binding-adapter.md), [devtools utils](./04-browser-cdp/devtools-utils.md) |
| 05 | Formatters | [↗](./05-formatters/README.md) | [console](./05-formatters/console-formatter.md), [network](./05-formatters/network-formatter.md), [issue](./05-formatters/issue-formatter.md), [snapshot](./05-formatters/snapshot-formatter.md), [heap](./05-formatters/heap-snapshot-formatter.md) |
| 06 | Performance & memory engine | [↗](./06-performance-memory/README.md) | [trace processing](./06-performance-memory/trace-processing.md), [heap snapshot mgr](./06-performance-memory/heap-snapshot-manager.md), [vendored workers](./06-performance-memory/third-party-workers.md) |
| 07 | Telemetry & logging | [↗](./07-telemetry/README.md) | [Clearcut logger](./07-telemetry/clearcut-logger.md), [watchdog](./07-telemetry/watchdog.md), [metrics](./07-telemetry/metrics-registry.md), [flags/persistence](./07-telemetry/flags-and-persistence.md), [logging](./07-telemetry/logging.md) |
| 08 | Utilities & shared types | [↗](./08-utilities/README.md) | [files/pagination](./08-utilities/files-and-pagination.md), [id/string/keyboard](./08-utilities/id-string-keyboard.md), [shared types](./08-utilities/shared-types.md) |
| 09 | Build & tooling | [↗](./09-build-tooling/README.md) | [build/bundle](./09-build-tooling/build-and-bundle.md), [codegen](./09-build-tooling/code-generation.md), [scripts](./09-build-tooling/scripts-reference.md), [config](./09-build-tooling/config-files.md), [lighthouse vendoring](./09-build-tooling/lighthouse-vendoring.md), [eval/metrics](./09-build-tooling/eval-and-metrics.md), [CI/release](./09-build-tooling/ci-and-release.md) |
| 10 | Testing | [↗](./10-testing/README.md) | [architecture](./10-testing/test-architecture.md), [fixtures/server](./10-testing/fixtures-and-test-server.md), [snapshots](./10-testing/snapshots.md), [e2e](./10-testing/e2e.md) |
| 11 | Skills, docs & integrations | [↗](./11-skills-docs/README.md) | [agent skills](./11-skills-docs/agent-skills.md), [docs](./11-skills-docs/docs-reference.md), [integrations](./11-skills-docs/integrations.md) |
| 12 | Conventions | [↗](./12-conventions/README.md) | [design principles](./12-conventions/design-principles.md), [TS rules](./12-conventions/typescript-rules.md), [contributing/release](./12-conventions/contributing-and-release.md) |

## Orientation tips
- **The hub is [`McpContext`](./03-context-state/mcp-context.md).** A graph analysis
  of this repo found it bridges ~16 subsystems. Read it before any non-trivial change.
- **A lot of code is vendored.** Trace/Lighthouse processing is a minified
  `chrome-devtools-frontend`/`lighthouse` bundle, not first-party. Layer 06 and
  [lighthouse-vendoring](./09-build-tooling/lighthouse-vendoring.md) mark the boundary.
- **Some docs are auto-generated.** `docs/tool-reference.md` and
  `docs/slim-tool-reference.md` carry "DO NOT EDIT" headers; they're regenerated by
  [code-generation](./09-build-tooling/code-generation.md).

## Provenance
This library was bootstrapped by fanning out one subagent per layer to read the
real source and write these docs. It is a derived artifact: when in doubt, the
code is the source of truth. Keep it honest — see [MAINTENANCE.md](./MAINTENANCE.md).
