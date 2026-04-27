# sst/opencode PR #24553 — fix(session): harden shell cancellation

- Link: https://github.com/sst/opencode/pull/24553
- Head SHA: `77307ad964920289f9e58f8745b4bdedff3a12fe`
- Size: +151 / -141 across 2 files

## Summary

Reworks the runner's shell-cancellation contract so an interrupt of a foreground shell is no longer indistinguishable from an unrelated fiber interrupt. Introduces a `cancelled: Deferred.Deferred<void>` on `ShellHandle` that `stopShell` flips before calling `Fiber.interrupt(shell.fiber)`, and the await arm now invokes `onInterrupt` only when the deferred is set OR the cause is interrupt-only (`runner.ts:155-163`). Also rewrites `SessionPrompt.shellImpl` (`session/prompt.ts:720+`) inside an `Effect.uninterruptibleMask` so the bookkeeping (user message, assistant message, tool part) is materialized atomically before any cancellable work runs.

## Specific-line citations

- `runner.ts:18-21`: `ShellHandle` gains `cancelled: Deferred.Deferred<void>` alongside `fiber`, which is the right shape because it lets the await arm distinguish cooperative cancellation from external interrupt without inspecting `Cause` shape alone.
- `runner.ts:107-110` + `runner.ts:155-163`: `stopShell` now `Deferred.succeed(shell.cancelled, undefined)` before `Fiber.interrupt`; the await branch checks `(yield* Deferred.isDone(cancelled)) || Cause.hasInterruptsOnly(exit.cause)` so a benign internal interrupt that happens to leave only interrupt causes still routes to `onInterrupt`, but a *failure* cause now correctly propagates via `Effect.failCause(exit.cause)` instead of being swallowed.
- `runner.ts:60-62`: `awaitDone` helper extracts the `Effect.catchTag("RunnerCancelled", …)` pattern that was inlined three times in `ensureRunning`'s `case` arms — each `Deferred.await` is now wrapped uniformly so the cancelled-deferred surface is the single source of truth.
- `session/prompt.ts:720+`: the entire `shellImpl` body is reshaped into `Effect.uninterruptibleMask((restore) => …)` with the message-bookkeeping block moved into a non-restored `Effect.gen` and only the spawned-process `runForEach` left interruptible — this is the correct shape for the "register before you start, never half-register" invariant that the bug presumably violated.

## Verdict

**merge-after-nits**

## Rationale

The `cancelled` deferred plus `uninterruptibleMask` framing is the right architectural change: the prior `Cause.hasInterruptsOnly` check was a heuristic that conflated user-driven cancellation, runtime-driven cancellation, and panic-induced interrupt, and the rewrite gives `stopShell` the explicit signal it needs. The `Effect.die(new Cancelled())` fallback at `runner.ts:160-162` correctly preserves fail-loud semantics when the cancellation arrives without an `onInterrupt` handler, which is the right defensive default.

Three nits before merge: (1) the new `awaitDone` at `runner.ts:60-62` swallows `RunnerCancelled` into `onInterrupt ?? Effect.die(e)` but the inline version it replaced threw on `Cancelled` not `RunnerCancelled` — the tag mismatch deserves either a comment or a regression test that pins the new tag is what `Cancelled` actually emits; (2) `Deferred.succeed(shell.cancelled, undefined).pipe(Effect.asVoid)` will silently no-op if `cancelled` was already set (e.g. double-stopShell from concurrent interrupts) and that idempotence isn't called out anywhere; (3) the `session/prompt.ts` shell-impl rewrite is large enough that a unit test pinning "abort-mid-stream → assistant message has `time.completed` set AND tool part is `completed` AND `aborted` metadata text is appended" would prevent regressions during the next refactor of this hot path.

