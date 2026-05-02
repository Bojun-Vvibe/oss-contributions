---
repo: openai/codex
pr: 20672
head_sha: a26a3d2095b413f59f3cb1cc3de3aa576cdc412f
title: "core: escalate repeated auto-review denials to user approval"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: openai/codex#20672 — `escalate repeated auto-review denials to user approval`

**Head SHA:** `a26a3d2095b413f59f3cb1cc3de3aa576cdc412f`
**Stat:** +283 / −191 across ~10 files in `codex-rs/core/src/{guardian,session,state,tasks,tools}/...`.

## Behavior change

Today, when the auto-review guardian rejects too many approval requests in
one turn, the session calls `abort_turn_if_active(.., TurnAbortReason::Interrupted)`
and the turn dies abruptly. After this PR, the same circuit-breaker trip
instead routes the *current* request to the human user via the existing
manual-approval flow.

The user-facing warning in `codex-rs/core/src/guardian/review.rs` is rewritten
accordingly:

```diff
-                    "Automatic approval review rejected too many approval requests for this turn ({consecutive_denials} consecutive, {total_denials} total); interrupting the turn."
+                    "Automatic approval review rejected too many approval requests for this turn ({consecutive_denials} consecutive, {total_denials} total); requesting manual approval for the current request."
```

and the immediate `runtime_handle.spawn(abort_turn_if_active(...))` block
that followed is removed.

## Mechanism

Two new helpers on `GuardianRejectionCircuitBreaker` (mod.rs around the
`pub(crate)` re-exports near line 36–42 and the `take_pending_auto_review_escalation`
impl at lines 124–129):

- `reset_auto_review_rejection_circuit_breaker(session, turn_id)` — clears
  the per-turn denial counter once a manual approval has been sourced, so
  the next denial doesn't immediately trip again on stale state.
- `take_pending_auto_review_escalation(session, turn_id)` — atomic
  test-and-clear of the `interrupt_triggered` flag. The `std::mem::take`
  pattern is correct (returns previous bool, leaves `false` behind).

Callers in `mcp_tool_call.rs`, `tools/network_approval.rs`, and
`tools/runtimes/apply_patch.rs` consult `take_pending_auto_review_escalation`
and, when set, fall through to `review_approval_request` against the user
(rather than the auto-reviewer) for that one request.

## Assessment

- The one-shot semantics are right: the flag is consumed (not just read),
  so a single circuit-breaker trip yields exactly one user-escalated
  request, after which the breaker resets via
  `reset_auto_review_rejection_circuit_breaker`.
- This is a clear UX win — losing a whole turn because the auto-reviewer
  got cranky is exactly the kind of failure mode users complain about.
- `session/tests.rs` has new coverage for the escalation path; the diff
  size (191 deletions) is mostly the old "spawn abort task" code being
  ripped out of multiple call sites, which is the right cleanup.

## Nits

1. **Race window.** `take_pending_auto_review_escalation` and
   `reset_auto_review_rejection_circuit_breaker` are two separate
   `lock().await` calls. If a second tool call slips in between them
   (concurrent tool invocations within one turn — possible for
   `apply_patch` + `network_approval` in flight together), the second
   caller can observe the breaker still tripped, double-escalate to the
   user, and then both call `reset` on top of each other. Worth either
   (a) folding take + reset into a single locked critical section, or
   (b) explicitly documenting that escalation is serialized at a higher
   layer (the `tasks/mod.rs` changes hint this is true but it's not
   obvious from the helper signatures).

2. **User-facing string.** "requesting manual approval for the current
   request" double-uses "request"; a small wording polish like
   "...; falling back to manual approval for this one." would read
   better in the warning event.

3. **Telemetry.** No metric counter incremented when the escalation
   path fires. Useful to know how often production sessions hit this in
   the wild — would inform whether the circuit-breaker thresholds are
   set sanely.

## Verdict

**`merge-after-nits`** — the core behavior change is correct and a
genuine UX improvement; the test coverage is in place. Worth the
maintainer addressing nit #1 (locking) before landing because it's a
real concurrency hazard, even if rare. Nits #2/#3 are polish.
