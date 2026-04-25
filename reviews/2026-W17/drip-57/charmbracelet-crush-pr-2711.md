# charmbracelet/crush #2711 — fix(ui): refresh todo pills on every session update

- URL: https://github.com/charmbracelet/crush/pull/2711
- Head SHA: `564ccba3427264327c84eae20af45771049f7914`
- Verdict: **merge-as-is**

## What it does

The top-of-chat todo pills panel was going stale on most state
transitions. The `pubsub.Event[session.Session]` handler in
`internal/ui/model/ui.go` only re-rendered when a todo transitioned
from "no in_progress" to "has in_progress" — every other transition
(`pending → completed`, `in_progress → completed`, additions where
nothing starts in_progress, status edits, reorders, "all done") was
silently skipped. The panel then sat stale until an unrelated event
forced a redraw.

## Diff notes

`internal/ui/model/ui.go:580-597` replaces the gated
`!prevHasInProgress && hasInProgressTodo(...)` branch with three
unconditional steps that mirror the sibling
`pubsub.Event[message.Message]` handler:

```
// start spinner if a todo just entered in_progress while busy
if hasInProgressTodo(m.session.Todos) && m.isAgentBusy() && !m.todoIsSpinning { ... }
// stop spinner when agent is no longer busy
if m.todoIsSpinning && !m.isAgentBusy() { m.todoIsSpinning = false }
// always re-render
m.renderPills()
```

Three things this gets right:

1. Spinner start now also requires `isAgentBusy()` — previously the
   message branch had this check and the session branch did not, so the
   two could disagree.
2. Spinner *shutoff* logic moves into the session branch too. Before
   this PR the only path that turned the spinner off lived in the
   message-event handler, so a session-only completion event would
   leave a phantom spinner.
3. `renderPills()` runs unconditionally — matches the message branch
   and is the actual fix for the stale-panel bug.

## Risk surface

- One extra `renderPills()` per session event. Cheap; pills layout is
  tiny and already rebuilt on message events.
- The two latent issues called out in the PR ("pubsub broker drops on
  full buffer", "tool result discarded if parent not yet in chat") are
  correctly deferred — both are race-conditional and out of scope for
  a deterministic stale-panel fix.

## Why this verdict

Tight 11-line behavior fix that brings two near-identical event
branches into agreement. Lands clean.
