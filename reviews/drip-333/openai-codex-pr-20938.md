# openai/codex PR #20938 — [codex] Ignore stale background rate-limit refreshes

- **Link:** https://github.com/openai/codex/pull/20938
- **Head SHA:** `1d493d1a25c252b3de6df879ac0e22adf5b867c0`
- **Verdict:** `merge-as-is`

## What it does

Adds monotonic generation tagging to background rate-limit refreshes so an
older `account/rateLimits/read` reply that completes after a newer one cannot
overwrite the more-recent snapshot. Concretely:

- Two new `App` fields at `codex-rs/tui/src/app.rs:511-512`:
  `next_rate_limit_refresh_generation: u64` and
  `latest_applied_rate_limit_refresh_generation: Option<u64>`.
- `App::refresh_account_rate_limits` (`app/background_requests.rs:43-48`)
  reads-and-bumps the next generation *before* spawning the tokio task, so
  the captured value is the request-creation order, not the
  reply-arrival order.
- `AppEvent::RateLimitsLoaded` grows a `refresh_generation: u64` field
  (`app_event.rs:243-244`).
- New `handle_rate_limits_loaded` (`app/event_dispatch.rs:1899-1929`) uses
  `is_some_and(|latest| latest >= refresh_generation)` to discard stale
  snapshots, otherwise records the applied generation and pushes snapshots
  to the chat widget.

## Design analysis

- **Monotonic generation > timestamp.** A timestamp would race with system
  clock skew on long-running TUI sessions (laptop sleeps, NTP corrections);
  a per-process `u64` generation is the canonical primitive for ordering
  *requests* not events.
- **Read-bump-then-spawn ordering at `background_requests.rs:43-46`** is
  load-bearing: capturing the value *before* `tokio::spawn` means even
  immediate `next` calls observe a higher generation, so an older spawn
  whose future hasn't yet polled is correctly identified as stale on
  eventual completion.
- **`saturating_add(1)` at line 45** makes the wraparound cliff at u64::MAX
  a no-op rather than a panic. In practice 2^64 refreshes will never happen,
  but the choice is consistent with the pending-write counters elsewhere in
  the file and avoids a theoretical correctness footgun.
- **Status-command `request_id` decoupled from staleness check** at
  `event_dispatch.rs:1922-1925`: even if the snapshot itself is stale, the
  user-initiated status command still gets its `finish_status_rate_limit_refresh`
  callback so the spinner unsticks. That's the right asymmetry —
  staleness is a *content* property, completion is an *interaction* property,
  they should not be conflated.

## Test

`app/tests.rs:125-138` is the dispositive test: applies generation 2, then
generation 1, asserts the recorded generation stays at 2. Order-independent
truth. Three `make_test_app` helpers updated in lockstep
(`test_support.rs:62-63`, `tests.rs:3796-3797`, `tests.rs:3859-3860`).

## Risks / nits

1. The error branch at `event_dispatch.rs:1917-1919` correctly does *not*
   advance `latest_applied_rate_limit_refresh_generation` so a transient
   failure doesn't poison subsequent in-flight generations. But it also
   doesn't *decrement* `next_rate_limit_refresh_generation` so the
   generation space sparsifies on errors — fine for ordering, worth a
   one-line comment.
2. `is_some_and(|latest| latest >= refresh_generation)` at
   `event_dispatch.rs:1908` uses `>=` which means a duplicate refresh with
   the same generation (impossible by construction here — each `spawn` gets
   a fresh generation) is also discarded. Belt-and-suspenders, harmless.
3. No assertion on the negative-property: that an older generation does not
   call `chat_widget.on_rate_limit_snapshot`. The test asserts the
   end-state generation but not the side-effect was skipped. A spy on
   `on_rate_limit_snapshot` would lock that.

## What I learned

The "spawn-time generation capture" pattern is the canonical fix for any
fire-and-forget background reload. Pair it with a single
`handle_X_loaded(generation, …)` choke point that does the staleness
compare in one place — the prior code at `event_dispatch.rs:651-676`
(deleted) had the snapshot-application logic *inline* in the event match,
so adding staleness would have required threading the check through both
the success and error arms. Pulling it into a method first, then adding
the generation, is the right order of operations.
