# charmbracelet/crush PR #2709 — fix(agent,ui): persist terminal finish for Run and Summarize + scoped spinner stall-guard

- Head SHA: `cd49de7de47f09a632e8845a6a6cbfa7ac58290f`
- Size: +470 / -15, 4 files

## Specific refs

- `internal/agent/agent.go:60-72` — new `summarizeRequestKeySuffix` constant + `summarizeRequestKey(sessionID)` helper. Fixes a real bug noted in the PR: `Summarize` previously registered its cancel func under bare `sessionID` while `Cancel` looked it up under `sessionID + "-summarize"` — the summarize cancel was effectively unreachable.
- `internal/agent/agent.go:443-447` — `Run`'s success-path `OnStepFinish` callback switched from `genCtx` to parent `ctx` for the terminal `Update`. Code comment explicitly calls out that fantasy swallows the error returned from this callback, so a failed Update would silently leave the spinner running.
- `internal/agent/agent.go:592-602` — after `Stream` returns nil, calls new `persistTerminalFinish(...)` defense-in-depth helper with `FinishReasonEndTurn`. Logs a `Warn` when it actually has to intervene, preserving observability of the underlying root-cause class.
- `internal/agent/agent.go:651-672` — `persistTerminalFinish` uses `context.Background()` for the write (correct: terminal bookkeeping must not be tied to a possibly-cancelled request ctx), no-ops on nil/already-finished messages.
- `internal/agent/agent.go:691-693, 745-758` — `Summarize` now registers under `summarizeKey`, deletes via detached ctx on cancel, and routes both error and success terminal-state writes through `persistTerminalFinish`.
- `internal/agent/agent.go:1167-1170` — `CancelAll` strips the suffix before recursing into `Cancel(sessionID)` — fixes a second bug where suffixed keys were treated as session IDs and missed the actual run cancel func.
- `internal/ui/chat/assistant.go` (+39 / -2) — 5-minute stall-guard scoped to `IsSummaryMessage == true` only. One-shot `slog.Warn` on trip. Normal assistant streams (which can legitimately run minutes on reasoning models) deliberately untouched.
- `internal/agent/busy_test.go` (+204) and `internal/ui/chat/assistant_test.go` (+148) — `TestIsSessionBusyAcrossKeys`, `TestCancelCancelsBothRunAndSummarize`, `TestCancelAllTranslatesSummarizeKeyToSessionID`, `TestEnsureAssistantFinishedNoOpWhenAlreadyFinished`, `TestEnsureAssistantFinishedNilIsSafe`, `TestAssistantSetMessageResetsStallLogged` — all the failure modes called out in the PR body have direct test coverage.

## Assessment

Excellent root-cause analysis with three independent fixes that compose well: (1) cancel-key namespacing is a real correctness bug that's been latent for a while — the named-constant + helper approach is right and the test for `CancelAll` suffix-translation is exactly the regression guard you want; (2) detached-context terminal writes via `persistTerminalFinish` is the correct pattern (a finish-write must outlive request cancellation); (3) scoping the UI stall-guard to summary-only avoids false-positives on slow reasoning streams.

The deliberate decision to keep the UI guard around even after fixing both root causes is the right call — it's a 5-minute safety net for a future regression, with the `Warn` log preserving root-cause visibility. Test coverage is thorough and the failure modes are named explicitly in test names. Only nit: the `slog.Warn` at agent.go:597-599 fires *before* `persistTerminalFinish` checks `IsFinished()` itself, so it logs even in the no-op case — invert the check or move the log inside the helper.

verdict: merge-after-nits
