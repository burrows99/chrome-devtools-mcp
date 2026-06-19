# CDP connection adapter

> **Layer:** Browser & CDP integration · **Sources:** `src/devtools/DevToolsConnectionAdapter.ts` · **Verified at:** commit afa5622 (2026-06-19)

## Purpose
Make a Puppeteer connection look like a DevTools-frontend `CDPConnection`, so the vendored DevTools SDK can send CDP commands and receive CDP events through a Puppeteer-managed session instead of opening its own websocket.

## Key files
- `src/devtools/DevToolsConnectionAdapter.ts` — the single class `PuppeteerDevToolsConnection`.

## Key types & functions
- `PuppeteerDevToolsConnection implements DevTools.CDPConnection.CDPConnection` — the adapter.
  - `send(method, params, sessionId)` — routes a command to the Puppeteer session with that id; resolves `{result}` or `{error}` (never rejects).
  - `observe(observer)` / `unobserve(observer)` — register/unregister DevTools `CDPConnectionObserver`s that receive forwarded events.
  - private `#startForwardingCdpEvents` / `#stopForwardingCdpEvents` / `#handleEvent` — the event-forwarding plumbing.

## How it works
The constructor takes a Puppeteer **root** `CDPSession`, grabs its underlying `connection()`, and immediately starts forwarding events for that session. It subscribes to `CDPSessionEvent.SessionAttached` / `SessionDetached` on the root session to start/stop forwarding for child sessions. The comment notes the root CDP session sees *all* descendant `sessionattached` events regardless of nesting depth, so no recursive subscription is needed.

Event forwarding attaches a wildcard `session.on('*', handler)` listener per session (handlers tracked in `#sessionEventHandlers` keyed by session id). `#handleEvent` skips the two `SessionAttached`/`SessionDetached` lifecycle events and fans every other CDP event out to all registered observers as `{method, sessionId, params}`.

```ts
// send() refuses the root session and looks up the target by sessionId
if (sessionId === undefined) throw new Error('Attempting to send on the root session...');
const session = this.#connection.session(sessionId);
if (!session) throw new Error('Unknown session ' + sessionId);
```

`send` deliberately catches errors into `{error}` rather than throwing, matching the DevTools `CDPConnection` contract.

## Relationships
- **Depends on:** `puppeteer-core` types (`Connection`, `CDPSession`, `CDPSessionEvent`, `Handler`) and the vendored **`DevTools.CDPConnection`** interface — both via `src/third_party/index.ts`.
- **Used by:** [DevtoolsUtils.ts](devtools-utils.md) `DEFAULT_FACTORY`, which constructs one `PuppeteerDevToolsConnection` per page from a secondary `CDPSession` and hands it to `targetManager.createTarget(...)`.

## Gotchas & non-obvious details
- **Vendored-frontend coupling / inference:** `send` casts `method as any` because, per the in-file comment, the rolled protocol versions of puppeteer and DevTools "don't necessarily match." This is a deliberate type-erasure at the boundary.
- Sending on the root session is forbidden — DevTools must always target a concrete child session id. The MCP layer scopes everything to a page's secondary session for this reason.
- The wildcard listener forwards *raw* protocol events; lifecycle events (`SessionAttached`/`SessionDetached`) are filtered out so observers only see real CDP domain events.

## Update triggers
- The vendored `DevTools.CDPConnection.CDPConnection` interface changes (method signatures, observer shape).
- Puppeteer changes `CDPSession`/`Connection` event semantics or the `'*'` wildcard listener behavior.
