# PR #2575 — fix: correctly identify context fate in mcp createSession

- URL: https://github.com/charmbracelet/crush/pull/2575
- Author: smlx
- Head SHA: `b5754e2c49ab000797286627ffd7711ea72cac84`

## Summary

Replaces a hand-rolled `context.WithCancel` + `time.AfterFunc(timeout,
cancel)` construction in `mcp.createSession` with a standard
`context.WithTimeout`. The old construct couldn't distinguish a
deadline-driven cancel from a caller-driven cancel inside `maybeTimeoutErr`
— both surfaced as `context.Canceled` — so timeouts were misreported as
cancellations in the status UI. Also retires the now-redundant
`ClientSession` wrapper struct (which existed solely to carry the
`cancel` func through `Close()`) and the test that exercised its
cancel-on-close behavior.

## Specific callouts

- `internal/agent/tools/mcp/init.go:38-50` — `ClientSession` wrapper struct
  deleted in favor of using `*mcp.ClientSession` directly. The wrapper's
  only job was to call `cancel()` in its `Close()` override; with
  `context.WithTimeout` plus `defer cancel()` inside `createSession`, the
  cancel happens deterministically when `createSession` returns regardless
  of success path. Cleaner ownership story.
- `internal/agent/tools/mcp/init.go:334-389` — The actual fix:
  ```go
  // before
  mcpCtx, cancel := context.WithCancel(ctx)
  cancelTimer := time.AfterFunc(timeout, cancel)
  // ... three error paths each calling cancel() + cancelTimer.Stop()
  // ... happy path calling cancelTimer.Stop()
  return &ClientSession{session, cancel}, nil
  // after
  mcpCtx, cancel := context.WithTimeout(ctx, timeout)
  defer cancel()
  // ... error paths just return
  // ... happy path returns session
  return session, nil
  ```
  The `defer cancel()` is correct here because `mcpCtx` is only used
  inside `createSession` — neither `transport` nor `session` retains a
  reference to it after `Connect` completes. (Worth confirming via
  `mcp-go-sdk` docs that `ClientSession` doesn't lazily use the connect
  context for later RPCs; if it does, this PR introduces a subtle
  use-after-cancel. The deleted test
  `TestMCPSession_CancelOnClose` was the *only* defense against that
  contract changing.)
- `internal/agent/tools/mcp/init.go:412-422` — `maybeTimeoutErr` now
  branches on `context.DeadlineExceeded` (timeout) vs `context.Canceled`
  (cancel) cleanly:
  ```go
  if errors.Is(err, context.DeadlineExceeded) {
      return fmt.Errorf("timed out after %s", timeout)
  }
  if errors.Is(err, context.Canceled) {
      return fmt.Errorf("cancelled")
  }
  return err
  ```
  This is the load-bearing semantic improvement. Before, a deadline hit
  would propagate as `context.Canceled` (because the AfterFunc explicitly
  called the cancel func), so the user saw "cancelled" instead of "timed
  out after 30s". Now they see the right thing.
- `internal/agent/tools/mcp/init_test.go` deleted — 38 lines removed.
  The test verified that `ClientSession.Close()` cancels the captured
  context. With the wrapper gone, the test is meaningless. Acceptable
  to delete, but the loss of *any* test in this file is a regression of
  coverage for a subtle area. Replacement test recommendation:
  ```go
  func TestCreateSession_TimeoutSurfacedAsTimeout(t *testing.T) {
      // Mock a transport that blocks past the timeout.
      // Assert maybeTimeoutErr returns "timed out after Xms", not "cancelled".
  }
  ```
  This pins the semantic that the PR is actually about.
- `internal/agent/tools/mcp/{prompts,resources,tools}.go` — All
  `*ClientSession` parameters became `*mcp.ClientSession`. Mechanical and
  correct, no behavior change.

## Risks

- **Lifetime contract**: if `mcp.ClientSession` ever uses the connect
  context after `Connect` returns (e.g. for keepalive pings, reconnect
  attempts, or a long-poll subscription), the `defer cancel()` will pull
  the rug. The old code deferred cancel until `Close()`. Need explicit
  confirmation from the `modelcontextprotocol/go-sdk` API contract that
  the connect context is consumed only during the handshake. If unclear,
  hold on this PR.
- **Coverage regression**: deleting the only test in this file without a
  replacement leaves the timeout-vs-cancel semantic unprotected. Easy to
  fix; high value.
- **Status UI strings change** from "cancelled" (when previously a timeout
  hit) to "timed out after Xs" — that's the intended fix but anyone with
  log greps or metric labels keyed on the old wording will see breakage.

## Verdict

**Verdict:** needs-discussion

The fix is conceptually correct and the `WithTimeout` swap is the right
shape. Two open items must be resolved before merge: (1) confirmation
from the mcp-go-sdk that `ClientSession` does not retain the connect
context past `Connect`, otherwise `defer cancel()` is a use-after-cancel
hazard; and (2) replacement test that pins the deadline-vs-cancel
semantic the PR is actually fixing. Once both are addressed, this is
merge-as-is.
