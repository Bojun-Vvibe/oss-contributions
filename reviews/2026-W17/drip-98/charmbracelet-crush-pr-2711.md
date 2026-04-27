# charmbracelet/crush#2711 — fix(ui): refresh todo pills on every session update

- **Head**: `564ccba3427264327c84eae20af45771049f7914`
- **Size**: +11/-3 in `internal/ui/model/ui.go`
- **Verdict**: `merge-as-is`

## Context

The top-of-chat "todo pills" panel was only being re-rendered on a single transition — `pubsub.Event[session.Session]` where `prevHasInProgress=false && hasInProgressTodo(new)=true`. Every other transition (the most common one being `in_progress → completed`, plus pure pending→completed, additions where none start in_progress, reorders, content edits, and the final all-completed state) silently skipped `renderPills()`, leaving the panel stale until an unrelated event (new message, resize, spinner tick) happened to trigger redraw.

## Design analysis

The fix at `internal/ui/model/ui.go:580-600` mirrors the already-correct sibling branch handling `pubsub.Event[message.Message]` instead of inventing a new state machine:

```go
if m.session != nil && msg.Payload.ID == m.session.ID {
    m.session = &msg.Payload
    // Start the spinner if a todo just entered in_progress while the
    // agent is busy.
    if hasInProgressTodo(m.session.Todos) && m.isAgentBusy() && !m.todoIsSpinning {
        m.todoIsSpinning = true
        cmds = append(cmds, m.todoSpinner.Tick)
    }
    // Stop the spinner if the agent is no longer busy.
    if m.todoIsSpinning && !m.isAgentBusy() {
        m.todoIsSpinning = false
    }
    // Any todo change (added, completed, reordered, content edited)
    // must re-render the pills; otherwise the panel stays stale
    // until an unrelated event triggers a redraw.
    m.renderPills()
}
```

Three correctness deltas vs the old code:

1. **Spinner-start gate widened.** Old: `!prevHasInProgress && hasInProgressTodo(new)` — fires only on the rising edge. New: `hasInProgressTodo && isAgentBusy() && !todoIsSpinning` — fires whenever the predicate becomes true and we aren't already spinning, with `!todoIsSpinning` preventing double-Tick scheduling. The old code held an implicit invariant that `prevHasInProgress=false` at session-event time meant "spinner is off"; that's true today but fragile against any code that mutates `todoIsSpinning` from elsewhere.
2. **Spinner-stop branch added** at `:592-594`. Previously the shutoff lived only in the message-event branch (PR body confirms this), so a session that finishes work via a session-event path (e.g. last todo completed → agent goes idle, but no message lands) would leave the spinner ticking forever. This closes that.
3. **`renderPills()` unconditional** at `:599`. The dropped `m.updateLayoutAndSize()` was the only previous path to `renderPills()`. The new code calls `renderPills()` directly and lets the existing layout-driven render path handle anything that changes panel *size* (which is rare — pills count change is the only real geometry trigger, and that already routes through `renderPills` → layout invalidation when needed).

## Why this is right

Two design notes the PR doesn't quite spell out but matter:

- **`renderPills()` is the right primitive to call unconditionally.** It's a memo-rebuild over `m.session.Todos`, not a layout pass — calling it on every session event is O(n_todos) work, n bounded by typical agent todo lists (≤20). Free.
- **The deferred concerns are correctly out-of-scope.** PR body identifies two adjacent latent bugs (`pubsub/broker.go:103-109` silent drops on full 64-slot subscriber buffer; `ui.go:1057-1063` tool-result discarded if parent tool-call item not yet in chat) and explicitly defers them. Both are race-conditional and broader; this PR keeps a deterministic-bug fix tight.

## Test note

PR body says `go build ./...` clean and `go test ./internal/ui/... -count=1` green including existing pills/ui-model tests. The state-machine changes here would benefit from one new pinning test that fires three sequential `pubsub.Event[session.Session]` payloads (`add-pending`, `pending→in_progress`, `in_progress→completed`) and asserts `renderPills` was called 3× and `todoIsSpinning` ends `false` — but every existing test passes and the change is small enough that this shouldn't block.

## Verdict reasoning

This is the smallest possible expression of the right model: when the source-of-truth (session) updates, refresh the dependent view (pills) and recompute the spinner gate. The PR description's root-cause quote pulled the actual broken predicate verbatim from the file, the fix mirrors a known-good sibling, and the deferred-out-of-scope notes show the author looked around and made a deliberate scope decision rather than missing them.

## What I learned

"Fire view-refresh on rising edge of a derived predicate" is a tempting micro-optimization (you only need a redraw when the visible state changes!) but it almost always rots — the moment the state has more than two transitions or the predicate's inputs grow, you start losing intermediate redraws and chasing flicker. The well-behaved sibling branch in this file already used the right pattern (re-render unconditionally, gate the spinner separately); this PR is essentially "make the two branches symmetric."
