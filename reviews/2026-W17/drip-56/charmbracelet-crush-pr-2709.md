# crush #2709 — persist terminal finish for Run and Summarize + scoped spinner stall-guard

- URL: https://github.com/charmbracelet/crush/pull/2709
- Head SHA: `06c523a1e80d517a8d621f8d400009b5551e5e6a`
- Verdict: **merge-after-nits**

## What it does

Three commits, all targeting one symptom: the chat spinner gets stuck
forever because the assistant or summary message never persists a terminal
finish part. The PR has two root-cause fixes (Run, Summarize) plus a UI
stall-guard as defense in depth.

Specifically:

1. `Run` and `Summarize` were calling `a.messages.Update(genCtx, ...)` from
   the `OnStepFinish` callback. `genCtx` could be cancelled milliseconds
   after the stream completed, the Update would return ctx-cancelled,
   fantasy swallows the error, and the message is left without a terminal
   finish part.
2. New `persistTerminalFinish` helper appends a finish part with
   `context.Background()` so the spinner-stop signal lands even if the
   per-turn ctx is dead.
3. Activerequests gets a `summarizeRequestKeySuffix` namespace so a
   concurrent `Run` and `Summarize` on the same session can't clobber each
   other's cancel registration.

## Diff notes

- The `Update(ctx, *currentAssistant)` switch (was `genCtx`) inside the
  OnStepFinish closure in `Run` is the real fix — without it, the rest of
  the defense-in-depth is just hiding the bug. Good that both layers are
  here.
- `persistTerminalFinish` is correctly idempotent:
  ```go
  if msg == nil || msg.IsFinished() { return }
  ```
  — calling it after a successful in-stream finish is a no-op.
- Detached-ctx writes in this helper are right for terminal-state bookkeeping;
  losing the spinner-stop signal because someone hit Ctrl-C is exactly the
  bug being fixed.
- The `summarizeRequestKey(sessionID)` split is a real concurrency bug fix
  on its own — previously `cancelMu`-protected map keyed on bare sessionID
  meant Run's cancel handle would overwrite Summarize's (or vice versa).
  Worth its own commit, which the PR does have.

## Nits / asks

- The "defense in depth" `slog.Warn` in `Run` fires every time the assertion
  trips — but with the OnStepFinish fix in place, that path should now be
  unreachable. Consider promoting the warn to a "shouldn't happen, file a
  bug" message, or downgrading to `Debug`, so it doesn't become noise that
  hides a real future regression.
- Worth a regression test that simulates "stream completes successfully,
  caller cancels ctx 1ms later" and asserts the message reaches
  `IsFinished() == true`. The current diff doesn't appear to add one for
  the Run path — only behavioral assertions on the helper.
- The new `summarizeRequestKeySuffix` constant is a magic suffix appended
  by `summarizeRequestKey`. That works, but a tiny test on
  `summarizeRequestKey("abc") != "abc"` would lock the contract for
  reviewers reading just the cancel-map code in isolation.

## Why this verdict

The substance is correct on all three commits, with a real concurrency bug
fixed alongside the spinner-hang. The asks are about test coverage and log
volume, not correctness — `merge-after-nits` rather than `merge-as-is`
because the `slog.Warn` in particular will cause production noise once
the underlying race is fixed.
