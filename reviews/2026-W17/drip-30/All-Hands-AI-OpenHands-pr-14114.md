# All-Hands-AI/OpenHands#14114 — fix(runtime): shrink BashSession tmux pane from 1000x1000 to 256x50

- PR: https://github.com/All-Hands-AI/OpenHands/pull/14114
- Author: he-yufeng
- +10 / -2
- Base SHA: `fb98faf4acc8afb2df65db06053328c4dc43a637`
- Head SHA: `fe7bd40ce34711818175f3d7f393544eaba30343`
- State: OPEN

## Summary

Drops the tmux pane geometry the runtime creates for every
`BashSession` from `x=1000, y=1000` to `x=256, y=50`. The PR
description (echoed in the inline comment) reports that the
1000×1000 grid causes the tmux server to consume ~80% of a CPU
core in kernel-mode pty I/O even when the session is detached and
the agent is idle waiting on inference. The choice of 256×50
keeps lines wide enough for typical `git diff` / `grep` /
`make` output while collapsing the cell count from 1,000,000 to
12,800 (~78× reduction).

## Specific findings

- `openhands/runtime/utils/bash.py:235` (head SHA `fe7bd40c`) —
  the `start_directory=self.work_dir, kill_session=True` kwargs
  are unchanged; only `x` and `y` shrink. The accompanying
  comment correctly notes that `capture-pane -pS -` reads the
  full scrollback regardless of the live pane height, so a
  smaller `y` doesn't lose any historical output.
- The choice of `x=256` is conservative — it's wider than the
  typical 80/120/160 column terminal and should keep `git diff`
  hunks unwrapped in the captured output. A common alternative
  is `x=200` or `x=180`; `256` errs on the side of "agent never
  trips on artificial wrap." Good call.
- `y=50` is the only knob with downside risk. If any tool
  emits a *redrawing* progress UI (npm progress bar, pip
  download, certain Docker output) the smaller pane will
  capture fewer in-flight redraw frames before the next one
  overwrites them. For OpenHands' use case (final-state capture
  via `capture-pane -pS -`) this is fine, but worth verifying
  that no test asserts on a mid-stream tmux frame.
- The CPU symptom (1000×1000 pty burning ~80% of a core idle)
  is consistent with known tmux behavior on grids that exceed
  what most terminals can render efficiently — tmux keeps a
  cell-state matrix and runs periodic redraw cost calculations
  proportional to cell count. The fix is exactly the right
  mitigation.

## Risks

- If an agent invokes a tool that *requires* a wide pane for
  its own column rendering (`docker ps --no-trunc`, `aws
  ec2 describe-instances` table output), 256 is still
  generous, but a future PR adding a tool that does
  `tput cols`-aware rendering may want to pass through.
- No test included. A simple assertion on `session.pane_width`
  or capturing CPU-time-per-second of the tmux subprocess in a
  smoke test would lock the regression in. Optional.

## Verdict

`merge-as-is`

## Rationale

Tiny, well-explained, well-justified one-line behavior change
with a load-bearing comment that future readers will appreciate.
The performance win is large (~80% of a core per idle session)
and the only theoretical downside (mid-stream tmux frame
capture) doesn't apply to OpenHands' final-state capture
pattern. Ship it.

## What I learned

When you give tmux a pty grid that's two orders of magnitude
larger than what any human terminal renders, the server's
periodic cell-state reconciliation work scales with cell count
even when nothing is happening in the pane. "Bigger is safer"
is wrong for pty geometry.
