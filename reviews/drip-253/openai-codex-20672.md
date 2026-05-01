# openai/codex #20672 — core: escalate repeated auto-review denials to user approval

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20672
- **HEAD SHA:** `a26a3d2095b413f59f3cb1cc3de3aa576cdc412f`
- **Author:** won-openai
- **Verdict:** `merge-after-nits`

## What the diff does

Replaces the "auto-review rejection circuit-breaker aborts the entire
turn" behavior with "escalate the *current* approval request to manual
user approval and reset the breaker on resolution". Five-file change:

1. `core/src/guardian/review.rs:202-216` — drops the
   `runtime_handle.spawn(...) abort_turn_if_active(...)` block in
   `record_guardian_denial` and rewords the warning from "interrupting
   the turn" to "requesting manual approval for the current request".
   Adds two new helpers — `reset_auto_review_rejection_circuit_breaker`
   (`:218-228`) clears the per-turn denial counter, and
   `take_pending_auto_review_escalation` (`:230-239`) reads + clears the
   breaker's `interrupt_triggered` flag in one atomic op.

2. `core/src/guardian/mod.rs:121-126` — new
   `take_pending_auto_review_escalation` on the breaker uses
   `std::mem::take` on `turn.interrupt_triggered`, the
   read-and-clear-in-one-step pattern that matters here because two
   parallel approval paths could otherwise both observe the flag and
   both escalate (then both reset).

3. `core/src/mcp_tool_call.rs:961-1077` — at
   `maybe_request_mcp_tool_approval`, when guardian routing returns
   `ReviewDecision::Denied` *and*
   `take_pending_auto_review_escalation(...)` returns true (i.e. the
   breaker tripped), the function falls through to the manual approval
   prompt path instead of applying the denial; on the user's response
   it resets the breaker via
   `reset_auto_review_rejection_circuit_breaker` at `:1050` and `:1074`
   so subsequent auto-reviews on the same turn start from a clean
   counter.

4. `core/src/session/mod.rs:1990-2105` — same shape applied to
   `request_permissions`. The `escalated_from_guardian` flag gates the
   guardian early-return; when set, the function falls through to the
   user-prompt path that already existed for non-guardian-routed
   sessions, then resets the breaker after the user responds.

5. `tests/suite/guardian_review.rs` updates lock the new behavior
   (existing escalation-aborts-turn assertion replaced with
   escalation-prompts-user-then-resets-breaker).

## Why it's right

The old "abort the turn after N denials" behavior was the wrong
forcing function — a guardian model that's wrong about a tool call
shouldn't be allowed to terminate productive work; it should be
allowed to *fail open* into the path that already exists for
sessions where guardian routing is off (manual user approval). The
escalation pattern keeps the safety property (no auto-approval after
repeated denials) while removing the unrecoverable-turn footgun.

The `std::mem::take`-based read-and-clear in
`take_pending_auto_review_escalation` (`mod.rs:122-125`) is the
load-bearing concurrency detail — two approval paths racing on the
same turn id would otherwise both observe `interrupt_triggered=true`,
both escalate, and both call `reset_*`, but each is independent and
the manual prompt is per-call so duplicating the prompt is the
benign degradation (vs. losing the escalation entirely if the flag
were checked then cleared in two separate calls).

The reset on user response (`:1050,1074` and the parallel
`session/mod.rs` call sites) is what closes the loop: if the user
approves, the breaker counter resets so the next auto-review denial
sequence starts fresh; if the user denies, the breaker still resets
because the user's "no" is itself a fresh authoritative signal that
supersedes the prior auto-rejection streak. Both interpretations are
defensible.

The sequencing — `escalated_from_guardian = matches!(decision,
Denied) && take_pending_auto_review_escalation(...)` at
`mcp_tool_call.rs:973-974` — only escalates on the
specifically-Denied path, so `Approved`/`ApprovedForSession`/
`ApprovedExecpolicyAmendment`/`NetworkPolicyAmendment` all still
short-circuit the user-prompt path. That's correct: the breaker
trips on consecutive denials, so a single approval naturally resets
the streak via `clear_turn` (called elsewhere on approve paths) and
the escalation guard never fires.

## Nits

1. **`session/mod.rs:1988+` clones `args.reason` into
   `request_reason` and `call_id` into a separate binding** to enable
   the fall-through path (the original code moved `args.reason` and
   `call_id` into the guardian-only event). The pattern is correct
   but the diff also captures `request_reason` *before* the guardian
   match — if guardian path resolves with `Approved`, `request_reason`
   is dropped without being used, costing one `String` clone per
   guardian-routed permission request. Minor allocation but worth a
   comment, or restructure to clone only on the escalation arm.

2. **`reset_auto_review_rejection_circuit_breaker` is called twice in
   `mcp_tool_call.rs`** (`:1050` for the elicitation path,
   `:1074` for the regular approval path). If a future reorganization
   merges the two paths, the reset call could be missed on one. A
   `defer!`-style guard or a single reset at the top of the
   post-prompt completion path would be more robust against drift.

3. **No locking test for the "escalation, user approves, *next*
   auto-review denial sequence starts fresh"** behavior. The current
   tests cover the escalation prompt + reset semantics for one
   sequence, but the most important property of the design (breaker
   state actually clears so subsequent denials trigger their own
   escalation rather than a permanent "never auto-deny again" state)
   needs a multi-cycle test: deny×N → escalate → user approves → run
   again → deny×N → escalate again. Without this, a future change
   that accidentally turns "reset" into "permanently disable
   breaker for this turn" would not be caught.

4. **`GuardianRejectionCircuitBreaker::take_pending_auto_review_escalation`
   uses `is_some_and(|turn| std::mem::take(&mut turn.interrupt_triggered))`**
   which is fine, but doesn't return the previous value — callers can
   only learn "was it set?" through the return type. For diagnostic
   logging it would help to also expose the per-turn `consecutive_denials`
   count *at the moment of escalation* so the warning event message
   could include the actual N (the reword at `:202-205` already
   surfaces this in the warning string, but only because
   `record_guardian_denial` interpolates locally — the escalation
   path can't see the count).

5. **The reword from "interrupting the turn" to "requesting manual
   approval for the current request"** at `:204` should also include
   forward guidance — something like "you can deny again to abort
   the call" or "approving will let the model continue from here" so
   the user knows what the prompt that's about to appear means
   relative to the warning.

## Verdict rationale

Right diagnosis (auto-review-aborts-turn was the wrong default), right
mechanism (escalate to the manual-approval path that already exists,
reset breaker on resolution), right concurrency primitive
(`std::mem::take` for atomic read-and-clear). Allocation cost in the
common-case approval path, two reset call sites that could drift,
missing multi-cycle test for the breaker-actually-resets property.

`merge-after-nits`
