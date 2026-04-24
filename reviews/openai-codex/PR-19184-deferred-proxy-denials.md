# PR-19184 — fix: handle deferred network proxy denials

[openai/codex#19184](https://github.com/openai/codex/pull/19184)

## Context

When the managed network proxy + Guardian/auto-review is enabled, a
network request can be approved synchronously (or deferred) at the
moment the command starts, but then later denied by the proxy after
the process is already running. Before this PR, the shell + unified
exec runtimes had no signal channel for an *async* proxy denial — the
command would either keep running past a denied request or
unregister the approval without surfacing the denial as the
command's failure.

The fix adds a `CancellationToken` to both `ActiveNetworkApproval`
and `DeferredNetworkApproval` in `codex-rs/core/src/tools/network_
approval.rs`. `record_call_outcome` only writes for *active*
registrations and now `cancellation_token.cancel()`s the owning call
after recording. A new `ExecExpiration::TimeoutOrCancellation { timeout,
cancellation }` variant in `codex-rs/core/src/exec.rs` lets callers
combine the regular command timeout with the network-denial token via
`with_cancellation(...)`. `cancel_when_either` spawns a small task
that cancels a derived token when either input cancels — this is the
plumbing that lets the shell runtime + unified-exec poll a single
unified expiration.

`unified_exec/process_manager.rs` (+159) stores the deferred approval,
terminates tracked processes on proxy denial, and returns the denial
as the process failure. `command_exec.rs` in the app server gains a
matching arm for the new expiration variant.

## Strengths

- **State consolidation in `NetworkApprovalCallState`.** Replacing
  two separate `Mutex<IndexMap>` + `Mutex<HashMap>` fields with a
  single `Mutex<NetworkApprovalCallState>` removes an entire class
  of "took one lock, then the other" ordering bugs. Outcome
  recording and active-call lookup are now atomic against each
  other.
- **Outcome only recorded for active calls.** The new guard in
  `record_call_outcome` — `let Some(call) = calls.active_calls.get
  (registration_id).cloned() else { return; }` — prevents stale
  proxy denials (arriving after the command already finished) from
  leaving phantom outcomes that the next call would inherit.
- **Cancellation precedence preserved.** A
  `NetworkApprovalOutcome::DeniedByUser` already in the map is not
  overwritten by a later policy denial; user denial wins. That's
  the right precedence.
- **`finish_call` returns a typed `Result<(), ToolError>`** so the
  caller can distinguish "user rejected" from "policy denied
  ($message)" from "no denial recorded".
- Extensive test additions: 153 lines added to `tests/suite/
  unified_exec.rs`, plus 49 added in `network_approval_tests.rs`.

## Concerns / risks

- **`cancel_when_either` leaks a tokio task** when neither input
  ever cancels. The spawned task `tokio::spawn(async move {
  tokio::select! { ... } cancel.cancel(); })` parks forever on the
  `cancelled()` futures. For long-lived sessions with many short-
  lived combined tokens, the task count climbs. If `first` and
  `second` both go out of scope and neither ever cancels (e.g. the
  command completes normally and nobody cancels the parent token),
  the spawned task is parked indefinitely. Consider either using
  `tokio_util::sync::CancellationToken`'s built-in
  `child_token()` / `drop_guard` patterns (a child token cancels
  when its parent does, no extra task needed), or bounding the
  spawned task with a shutdown signal tied to command completion.
- **`TimeoutOrCancellation` is duplicated across two crates.** The
  same `tokio::select!` over `tokio::time::sleep(timeout)` and
  `cancellation.cancelled()` lives in both `core/src/exec.rs::
  ExecExpiration::wait` and `app-server/src/command_exec.rs::run_
  command`. The app-server file imports `ExecExpiration` from
  `codex-core` already; the second copy can call
  `expiration.wait().await` instead of pattern-matching the variant
  itself. As written, a future change to one branch will silently
  diverge from the other.
- **`with_cancellation` collapses `Cancellation` into another
  combined token but then re-wraps it as
  `ExecExpiration::Cancellation(cancel_when_either(...))`** rather
  than `TimeoutOrCancellation`. Mostly fine, but it means the
  `wait_ms` accessor will return `None` for that case — callers
  reading "what's my timeout?" lose the original signal. Not a bug
  per se; just an asymmetry to document.
- **`is_cancelled()` on `DeferredNetworkApproval` is a snapshot.**
  Callers in unified-exec poll it; if a denial arrives between the
  poll and the next check, the gap can be up to one polling
  interval. The `cancellation_token()` accessor exists for `await`-
  ing properly, but the `is_cancelled()` API invites the wrong
  pattern. Consider hiding it or commenting why polling callers are
  acceptable.
- **`take_call_outcome` is now `#[cfg(test)]`-only.** The new
  `remove_call` returns the outcome and is the production path.
  Worth adding a comment noting this is the consolidated production
  API and `take_call_outcome` is a legacy test helper, otherwise a
  reviewer will wonder why the same logic exists twice.

## Verdict

**Request changes.** The state consolidation and end-to-end signal
plumbing are correct, but the `cancel_when_either` task leak is a
real concern in long-running session processes (every command spawns
a parked task that lives until *something* cancels). Use
`CancellationToken::child_token()` / `DropGuard` instead of a
spawned watchdog. Also dedupe the `TimeoutOrCancellation` match arm
between `core/src/exec.rs` and `app-server/src/command_exec.rs`
before merging.
