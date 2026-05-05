# google-gemini/gemini-cli PR #26484 — fix(core): prevent unhandled promise rejection on IDE MCP fetch failure

- Repo: `google-gemini/gemini-cli`
- PR: #26484
- Head SHA: `d161659c9d50`
- Author: `SAY-5`
- Scope: 1 file, +25 / -14
- Verdict: **merge-after-nits**

## What it does

Splits the existing `IdeClient.registerClientHandlers()` in `packages/core/src/ide/ide-client.ts` into two phases and **moves the `client.onerror` registration to fire *before* `client.connect(transport)`**, eliminating an unhandled-promise-rejection window during the SSE/HTTP transport handshake.

Concretely:

- New private method `registerTransportHandlers()` at `ide-client.ts:498-520` (formerly the `onerror` block that lived inside `registerClientHandlers`).
- `registerClientHandlers()` is left with just the notification handlers (`IdeContextNotificationSchema`, `IdeDiffAcceptedNotificationSchema`, etc.).
- Both connect paths — the SSE/HTTP path at `ide-client.ts:606-610` and the stdio command path at `ide-client.ts:640-644` — now invoke `this.registerTransportHandlers()` **before** `await this.client.connect(transport)`, and only invoke `this.registerClientHandlers()` after the connection resolves.
- The doc comment on `registerTransportHandlers` (line ~498) explicitly states the contract: *"Must be invoked before client.connect so that errors surfaced during connection setup or by a long-lived SSE stream are routed through onerror instead of becoming unhandled promise rejections."*

## Why this is right

- **Real bug, real fix.** MCP `client.connect(transport)` opens a long-lived SSE stream; if the transport throws after `connect` returns (e.g. server hangs up mid-stream, network hiccup, IDE process dies), the MCP SDK propagates that via `client.onerror`. If `onerror` isn't registered yet, Node logs an unhandled-rejection warning (and on `--unhandled-rejections=strict` exits the process). Registering `onerror` *before* `connect` plugs that hole.
- **Same fix at both connect call-sites.** Both the `StreamableHTTPClientTransport` branch (`ide-client.ts:600-612`) and the `StdioClientTransport` branch (`ide-client.ts:634-646`) get `registerTransportHandlers()`, so the bug is closed for both transports, not just the one in the bug report.
- **No behavioural change for notification handlers.** The notification handlers stay in `registerClientHandlers()` and are still installed *after* `connect`, which is fine because notifications can only arrive once the channel is open.
- **No new state, no new fields.** Pure code-motion + reordered call sites. Easy to review, easy to revert if it ever needs to be.

## Nits

1. **No regression test.** The fix is correct but is exactly the kind of timing-window bug that quietly regresses if someone later "consolidates" the two methods back into one. A unit test that mocks `client.connect` to reject and asserts (a) `client.onerror` was set before `connect` was called (e.g. via call-order spies) and (b) no unhandled rejection escapes the `IdeClient` boundary would lock the contract in.

2. **Comment lives on `registerTransportHandlers`, not on the call sites.** The two call sites at `ide-client.ts:606-610` and `:640-644` are where someone "tidying up imports" a year from now is likely to reorder things ("install all handlers in one place after `await connect`"). A one-line comment at each call site — `// MUST be before await connect — see registerTransportHandlers docstring` — would make the invariant locally visible.

3. **Method name slightly misleading.** `registerTransportHandlers` only registers `onerror`. If a future refactor adds `onclose`, `onmessage`, etc. to the transport phase, the name fits; today it'd be more honest as `registerOnErrorEarly()` or `registerTransportErrorHandler()`. Minor naming taste.

4. **Symmetry note (non-blocking).** If `registerTransportHandlers` is also safe to invoke a second time without effect, calling it again from inside `registerClientHandlers` (after connect) would make the wiring belt-and-braces idempotent. Currently it's only called from the connect paths. Not necessary, just defensive.

## Verdict
**merge-after-nits** — straightforward, narrowly-scoped fix to a real unhandled-rejection class of bug, and it covers both transports. Add a unit test or at least a call-site comment so the invariant ("transport error handler installed before connect") survives the next refactor.
