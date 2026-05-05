# Review: charmbracelet/crush PR #2575

- **Title:** fix: correctly identify context fate in mcp createSession
- **Author:** smlx
- **Head SHA:** `b5754e2c49ab000797286627ffd7711ea72cac84`
- **Files touched:** 5
- **Lines:** +15 / -69

## Summary

Refactors MCP session lifetime in
`internal/agent/tools/mcp/init.go`:

1. Drops the bespoke `ClientSession` wrapper (which held a
   `context.CancelFunc` alongside `*mcp.ClientSession`) and reverts
   the global `sessions` map to `csync.NewMap[string,
   *mcp.ClientSession]()`.
2. In `createSession`, replaces the
   `context.WithCancel(ctx) + time.AfterFunc(timeout, cancel)`
   pattern with the idiomatic `context.WithTimeout(ctx, timeout)`
   and a `defer cancel()`.
3. Updates `maybeTimeoutErr` (line ~412) to differentiate the two
   underlying errors:
   - `context.DeadlineExceeded` → `"timed out after %s"`
   - `context.Canceled` → `"cancelled"`
   This is the "correctly identify context fate" the PR title
   refers to: previously every `context.Canceled` was reported as a
   timeout, even when a parent context was actually cancelled by
   the user.
4. Deletes `internal/agent/tools/mcp/init_test.go` (38 lines) —
   the test was tightly coupled to the deleted `ClientSession`
   wrapper and no longer applies.

## Notes

- The wrapper was holding `cancel` precisely so it could be invoked
  on `Close()` — but the new code uses `defer cancel()` *inside*
  `createSession`, which means the context is cancelled the moment
  `createSession` returns. The session subsequently kept by the
  caller no longer has a cancellable context. Reviewers should
  verify `*mcp.ClientSession` either holds its own internal context
  for ongoing RPCs (so the deferred cancel only tears down the
  initial dial) or that there is no longer a need for a long-lived
  cancel handle. If the underlying SDK uses the passed `mcpCtx` for
  *post-init* operations, this PR could prematurely cancel them.
- The error-classification change is a clear win and would justify
  the PR on its own.
- Removing the test file with no replacement reduces coverage
  around session lifetime — would be good to land a small test
  that asserts the new error text for the deadline vs cancelled
  paths.

## Verdict

**needs-discussion** — the lifetime semantics around `defer
cancel()` versus the prior `cancelTimer.Stop()` + late-cancel-via-
wrapper need a maintainer to confirm the MCP SDK doesn't depend on
the dial context after `Initialize` returns. If it does, sessions
get a cancelled context for free. The error-classification half is
ready as-is.
