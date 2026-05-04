# sst/opencode PR #25670 — fix(mcp): auto-reconnect on transport errors (#25287)

- **Link:** https://github.com/sst/opencode/pull/25670
- **Head SHA:** `5c803b8db45c436e2dc1962a17ce3ce08e2b25d2`
- **Verdict:** `merge-after-nits`

## What it does

Wraps every MCP tool execution in a `.catch` that classifies the error via a
new `isTransportError(e)` predicate (`packages/opencode/src/mcp/transport-error.ts:38-48`)
and on transport failures triggers a single-flight `reconnectClient(name)`
(`src/mcp/index.ts:125-145`) before retrying the call against the freshly
restored client (`src/mcp/index.ts:167-180`). The previous `convertMcpTool`
(deleted at `src/mcp/index.ts:75-104`) had no recovery — an EPIPE on a long
running stdio MCP would surface as a hard tool error to the model and the
client stayed dead until process restart.

## Design analysis

- **Single-flight is the right primitive.** Multiple in-flight tool calls
  hitting EPIPE simultaneously should not each spawn a fresh `createAndStore`;
  the `reconnecting: Map<string, Promise<boolean>>` (line 114) coalesces them
  and `.finally(() => reconnecting.delete(name))` (line 140-142) guarantees
  the entry clears even on rejection so the *next* failure window can
  re-trigger. Without that delete, a single failed reconnect would
  permanently pin the entry.
- **Classifier shape is sound.** `transport-error.ts:48` covers Node/undici
  PascalCase Bun runtime codes, plus a 4xx/5xx pass on `StreamableHTTPError`
  with explicit 401/403 carve-outs (auth flow, not transport) and `code === -1`
  treated as transport because the SDK uses it for protocol-level breakage.
  The `err.cause?.code` chain at line 250 catches the wrapped-fetch shape
  undici emits.
- **Retry-once-only is correct.** A second transport failure after reconnect
  rethrows; no exponential ladder, no re-entrant reconnect. The model sees
  one degraded tool result instead of multi-second hangs.

## Risks / nits

1. `transport-error.ts:252` falls back to `err.message.includes("fetch failed")`
   string match. That string is undici-version-dependent; pin behind a comment
   noting which undici versions emit it, or prefer the `cause.code` path.
2. The `"Timeout"` and `Bun.SocketClosed`-style codes in the
   `TRANSPORT_ERROR_CODES` set (lines 232-235) overlap with legitimate
   user-controlled tool timeouts. A 30s tool timeout that fires because the
   *remote tool* is slow (not the transport) will now be reclassified as
   transport and trigger a reconnect cycle. Consider distinguishing
   "client-side timeout we set" (don't reconnect) from "socket timeout"
   (reconnect).
3. `reconnectClient` returns `boolean` but discards the error in the `.catch`
   at line 136-139. Worth threading the underlying error into the rethrow at
   `src/mcp/index.ts:175,178` so the model sees "reconnect failed: <reason>"
   not just the original transport error masquerading.
4. New CI matrix (`mcp-probe`) at `.github/workflows/test.yml:9-35` runs on
   Linux + Windows but only with a 2-minute timeout. The probe script is in
   `transport-error-probe.mjs`; if it's environmental (real socket) it will
   flake on slow runners.
5. The package.json at line 56-58 drops the trailing newline ("No newline at
   end of file") — house style nit.

## What I learned

The classifier-with-explicit-carve-outs pattern (`401/403 → not transport`)
is the load-bearing detail. A naive "any 4xx counts" classifier would
auto-reconnect on auth failures, papering over OAuth-token-expired scenarios
that the user needs to see in order to re-authenticate. The same retry
machinery that helps for sockets actively hurts for auth — which is why
401/403 are spelled out at `transport-error.ts:243-244` rather than relying
on the `>= 400` fall-through at line 246.
