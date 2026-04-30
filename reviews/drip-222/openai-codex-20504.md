---
pr-url: https://github.com/openai/codex/pull/20504
sha: 14d53a8f9797
verdict: needs-discussion
---

# fix flaky test falls_back_to_registered_fallback_port_when_default_port_is_in_use

One-line semantic flip at `codex-rs/login/src/server.rs:132` inside `ShutdownHandle::shutdown`: `self.shutdown_notify.notify_waiters()` becomes `self.shutdown_notify.notify_one()`. The PR title names the symptom (a flaky test about fallback-port behavior) but the diff doesn't include a test change, doesn't include a comment naming why the semantic flip is correct, and doesn't include a body explanation — so the only evidence the fix is right is the semantic difference between the two `tokio::sync::Notify` methods themselves.

The relevant Tokio contract: `notify_waiters()` wakes *all* tasks currently parked on `notified()` calls but does *not* leave a permit behind, so a future `notified()` call will park until the *next* `notify_*`. `notify_one()` wakes one parked task, and if no task is parked it leaves a permit behind so the next `notified()` returns immediately. The flake mode this fix is plausibly addressing is the "ShutdownHandle::shutdown is called *before* the server task reaches its `notified().await` call" race — under `notify_waiters()` that wake is lost (no permit stored), so the server task parks indefinitely; under `notify_one()` the permit is stored, and when the server task does call `notified().await` it returns immediately.

That's the textbook race, and the textbook fix is correct *if* the contract is "shutdown is called at most once and the server task's notified-await is the only consumer." But the PR ships zero evidence for either assumption. Concrete questions a reviewer needs answered before this is mergeable as-is:

(1) **Is shutdown actually called at most once?** `notify_one()` stores a single permit regardless of how many times it's invoked, so multiple shutdown calls collapse to one wake. That's fine for the shutdown semantic (idempotent shutdown is what you want), but if there's ever a second `notified().await` call elsewhere in the server lifecycle that's *not* meant to be a shutdown signal, it'll silently consume the stored permit and produce a different bug class. A grep for `shutdown_notify.notified()` call sites would settle this.

(2) **Is there only one waiter?** `notify_one()` wakes exactly one task. If the server's design assumes *all* tasks listening on the shutdown notify get woken (the prior `notify_waiters()` semantic), then `notify_one()` is a regression: only one of N waiters wakes, the rest park forever. The PR doesn't show the `notified().await` call site so a reviewer can't verify this.

(3) **What was the exact flake symptom?** "Flaky test" is the diff's only justification, but the test-name suffix (`...when_default_port_is_in_use`) suggests the test exercises a fallback-port-binding code path, not a shutdown code path directly. Did the flake reproduce the "shutdown-called-before-await" race, or some other race that this fix happens to suppress without addressing the root cause? Without a reproducer or before/after flake-rate number, the fix could be papering over a deeper issue.

(4) **Why no test?** A test that pins the contract — "shutdown called before notified-await still wakes the server within X ms" — would both prove the fix and guard against regression. The current shape leaves the next person to touch this file (e.g. someone adding a second `notified()` consumer) with no signal that the `notify_one` choice was load-bearing.

The change is plausibly correct and the diff is one line, so the cost of merging-as-is is small. But the cost of *waiting for an answer to question (2)* is also small (one grep), and the answer determines whether this is `merge-as-is` or `request-changes`. Hence: needs-discussion. If the author confirms there's exactly one `notified().await` consumer and shutdown is idempotent-by-design, this becomes `merge-as-is` with a one-line comment ("notify_one stores a permit so shutdown-before-await still wakes; do not flip back to notify_waiters without auditing notified() consumers").

## what I learned
The `Notify::notify_waiters` vs `notify_one` distinction is exactly the kind of "one-line fix that changes a contract" that needs a comment more than it needs a test, because the contract change is invisible at the call site. Two months from now, someone reading `notify_one()` in a shutdown handler will reasonably assume "one waiter, one wake" and might "fix" it back to `notify_waiters()` to support multiple waiters — re-introducing the original race. The defensive shape is to leave a comment naming the contract: "stores a permit so shutdown-before-await still wakes; assumes single waiter."
