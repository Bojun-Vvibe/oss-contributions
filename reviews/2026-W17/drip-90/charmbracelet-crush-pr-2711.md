---
pr: 2711
repo: charmbracelet/crush
sha: 564ccba3427264327c84eae20af45771049f7914
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2711 — fix(ui): refresh todo pills on every session update

- **Head SHA**: `564ccba3427264327c84eae20af45771049f7914`
- **Author**: KimBioInfoStudio
- **Size**: tiny (single hunk in `internal/ui/model/ui.go` `Update` for `pubsub.Event[session.Session]`)

## Summary
Reworks the todo-spinner + pill-render trigger on session updates so that:
1. Spinner starts when *any* todo is in_progress AND the agent is busy AND the spinner isn't already running (previously: only on the rising edge of "no-in-progress → has-in-progress");
2. Spinner stops when the agent is no longer busy;
3. `m.renderPills()` runs on every session update (previously: only when the spinner was newly started, behind `updateLayoutAndSize()`).

## Specific findings
- `internal/ui/model/ui.go:582-591` — old code only re-rendered the pill panel on the in-progress *rising edge*. Any other todo mutation (added, completed, reordered, content-edited) left the panel stale until an unrelated event triggered a redraw. The new unconditional `m.renderPills()` at the end of the session-event branch is the right fix and matches the comment ("Any todo change … must re-render the pills").
- `internal/ui/model/ui.go:583-586` — start condition `hasInProgressTodo(m.session.Todos) && m.isAgentBusy() && !m.todoIsSpinning` is correct *for entering steady-state spinning* but loses one edge case the old code arguably handled: if the user manually toggles a todo to in_progress while the agent is *not* busy (rare — usually only happens via the agent's tool calls), the spinner won't start. Old code also wouldn't have started in that case (it required the rising edge), so behavior is preserved. Acceptable.
- `internal/ui/model/ui.go:587-590` — stop condition `m.todoIsSpinning && !m.isAgentBusy()` — fires on *every* session update where the agent went idle. That means the spinner-tick `cmds` chain is no longer scheduled, but there's no explicit `cmds = append(cmds, ...)` to cancel an in-flight spinner tick. In bubbletea this is fine (`m.todoIsSpinning = false` makes the next tick handler short-circuit), but worth a comment noting the implicit cancellation pattern.
- `internal/ui/model/ui.go:594-596` — `m.renderPills()` is called unconditionally. Verify it's idempotent and cheap; if it allocates a new lipgloss tree on every call, this could be measurable on sessions with hundreds of todos. From the function name and its prior placement inside `updateLayoutAndSize()`, looks fine, but worth the maintainer flagging if it isn't.
- The removed call to `m.updateLayoutAndSize()` is concerning: previously, *adding the first in_progress todo* would re-run layout. New code only renders pills, not layout. If pills can change *height* (multi-row pill rendering when a long todo wraps), the surrounding panes' heights may now be wrong on the first in-progress todo until the next layout-triggering event. The PR title only mentions "refresh todo pills", so this is an undocumented semantics change. Either add `m.updateLayoutAndSize()` back when the pill count crosses a wrap threshold, or confirm pills are fixed-height.
- No new test. The `Update` function is hard to unit-test, but a quick check that `renderPills` is invoked once per session-event delivery would be cheap.

## Risk
Low for the documented "stale pills" fix. Medium for the silent removal of `updateLayoutAndSize()` — if pill height varies, neighboring panels miscalculate.

## Verdict
**merge-after-nits** — confirm pills are fixed-height (or restore the layout call when they grow), add a one-line comment about the implicit spinner-tick cancellation, and document the `updateLayoutAndSize()` removal in the PR body.
