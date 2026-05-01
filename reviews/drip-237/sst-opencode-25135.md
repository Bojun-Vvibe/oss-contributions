# sst/opencode#25135 — fix(opencode): reconnect MCP transport on session-expiration error and retry once

- **PR**: https://github.com/sst/opencode/pull/25135
- **Head SHA**: `57e2337c50b26fab9b080948e556976e32de2572`
- **Size**: +352 / -15, 2 files (`packages/opencode/src/mcp/index.ts`, `packages/opencode/test/mcp/session-recovery.test.ts`)
- **Verdict**: **merge-after-nits**

## Context

Remote streamable-HTTP MCP servers may invalidate `mcp-session-id` for any reason (server restart, idle timeout, periodic rotation). The SDK surfaces this as a wrapped `Error POSTing to endpoint: {"error":"Session not found"}`. Pre-fix, every subsequent `client.callTool` against that server failed until the user manually disconnected and reconnected. Closes #25137; orthogonal to the related `listTools()` eviction path (#17099) and the desktop session-restore crash (#25131).

## What's right

- **Detection at the right surface.** New exported `isSessionExpiredError(err)` at `packages/opencode/src/mcp/index.ts:128-131` matches three case-insensitive substrings (`session not found`, `invalid session`, `mcp-session-id`) on `err.message ?? String(err)`. Three patterns covers the SDK's wrapped-error shape plus the two adjacent variants other servers emit; staying string-based avoids coupling to any single transport's typed error class.
- **Closure freshness via resolver function.** `convertMcpTool` signature flips from `(mcpTool, client, timeout)` to `(mcpTool, serverName, getClient, reconnect, timeout)` at `:135-141`. The execute path now calls `getClient()` on every invocation rather than capturing a stale `client` reference, so once `reconnect()` swaps `s.clients[clientName]` the next tool call sees the fresh transport. This is the load-bearing structural fix — without it, even a successful reconnect would leave every existing `Tool` closure pinned to the dead client.
- **Bounded one-shot retry.** `:171-181` — initial call → on `isSessionExpiredError` → `reconnect()` → single retry on `fresh`. A fresh transport that fails again signals a real problem; the `return call(fresh) // single retry; do not loop` comment makes the no-loop policy explicit. Correct fail-loud direction.
- **Effect-context capture for re-entry.** `:670-672` captures `Effect.context<never>()` once at `tools` setup, then `reconnect` uses `Effect.runPromiseWith(ctx)(createAndStore(...))` at `:705` to re-enter Effect-land from the AI SDK's plain async `Tool.execute`. The comment on `:667-669` documents the boundary correctly.
- **Reconnect-failure surfacing.** `reconnect()` at `:702-712` returns `undefined` on `createAndStore` failure with a `log.warn("mcp reconnect failed", {server, error})` line, and the `execute` arm at `:178-180` re-throws the *original* error rather than the reconnect error — the right call for triage (the user sees the symptom, not the recovery attempt).

## Risks / nits

- **`isSessionExpiredError` substring-matches `mcp-session-id` anywhere in the message.** A user-controlled tool argument that the server echoes back inside an unrelated error message (e.g. a 400 with `"mcp-session-id was rejected"`) would trigger a reconnect. Low-impact (worst case: one wasted reconnect), but consider anchoring the regex to known SDK prefixes or matching only when the message also contains `POSTing to endpoint`.
- **No backoff between reconnect attempts on rapid-fire tool calls.** If the server is in a flapping state, two parallel `Tool.execute` calls can both detect the expiration and both call `reconnect()`. `createAndStore` is presumably idempotent on `s.clients[clientName]` write but a brief comment confirming the contract would help future readers.
- **`mock` injection ordering.** The test uses `void mock.module(...)` followed by `await import("../../src/mcp/index")` — works in Bun but worth noting the test file is the only consumer of `mock.module` in this directory; consider adding a comment that the dynamic import after-mocks is intentional.

## Verdict

**merge-after-nits.** Surgical, structurally correct fix at the right boundary (closure-resolved client, not captured client). 272 lines of test for 80 lines of production code is the correct ratio for a recovery path that's hard to repro in CI.
