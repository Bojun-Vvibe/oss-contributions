# charmbracelet/crush PR #2711 — fix(ui): refresh todo pills on every session update

- URL: https://github.com/charmbracelet/crush/pull/2711
- Head SHA: `564ccba3427264327c84eae20af45771049f7914`
- Files touched: 1 (UI session update handler)
- Verdict: **merge-after-nits**

## Summary
Reworks the todo-spinner / pill-render logic in the session update handler so
that (a) the spinner starts whenever a todo is in_progress *and* the agent is
busy, (b) the spinner stops as soon as the agent finishes, and (c) `renderPills()`
runs on *every* session update so the pill panel doesn't go stale when the only
change is a todo edit/reorder.

## Cited concerns

1. `internal/ui/model/ui.go:582-598` (post-patch): the previous
   `prevHasInProgress` edge-trigger is replaced with a level-trigger
   (`hasInProgressTodo && isAgentBusy && !todoIsSpinning`). That's the right
   semantics but it relies on `m.todoIsSpinning` being correctly cleared.
   The companion `if m.todoIsSpinning && !m.isAgentBusy()` block does the
   clearing — confirm there is no other code path that sets
   `todoIsSpinning = true` without going through this gate, otherwise the
   spinner can be left running after the agent stops.

2. `internal/ui/model/ui.go:597`: `m.renderPills()` now runs on every session
   update for the matching session ID. For long sessions with frequent
   keepalive-style updates this adds per-event work. If `renderPills()` is
   non-trivial (string building, lipgloss styling), consider gating on an
   actual diff (`!todosEqual(prev, msg.Payload.Todos) || titleChanged`) to
   avoid redundant renders. Probably fine in practice but worth measuring.

3. The old code called `m.updateLayoutAndSize()` when the spinner started.
   The new code does not. If pill height changes when the spinner is shown
   (extra frame/pad), the layout may now be one frame stale on the
   transition. Check whether `renderPills()` triggers a re-layout on its
   own; if not, restore the `updateLayoutAndSize()` call inside the
   spinner-start branch.

4. No test added; tea Update-handler unit tests do exist elsewhere in the
   crate. A small one asserting "in_progress while busy → spinner ticks; busy
   ends → spinner stops; unrelated todo edit → renderPills called" would
   lock this in.

## Verdict
**merge-after-nits** — correctness improvement over the edge-triggered
version. Verify the dropped `updateLayoutAndSize()` and add a focused test.
