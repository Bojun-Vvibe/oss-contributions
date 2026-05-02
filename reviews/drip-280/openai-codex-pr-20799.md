# Review: openai/codex #20799 — [codex] Add goal lifecycle metrics

- **PR**: https://github.com/openai/codex/pull/20799
- **Head SHA**: `4c0d47543b1daae36f5a830b0c8fbf8e37c144b8`
- **Diff size**: ~300 lines, primary file `codex-rs/core/src/goals.rs`

## What it changes

Wires OpenTelemetry counters and histograms for goal lifecycle events:

- New imports at `goals.rs:9-13`: `GOAL_BUDGET_LIMITED_METRIC`, `GOAL_COMPLETED_METRIC`,
  `GOAL_CREATED_METRIC`, `GOAL_DURATION_SECONDS_METRIC`, `GOAL_TOKEN_COUNT_METRIC` from
  `codex_otel`.
- `emit_goal_created_metric()` (`goals.rs:45-49`): bumps `GOAL_CREATED_METRIC` counter
  with no tags.
- `emit_goal_terminal_metrics_if_status_changed()` (`goals.rs:51-81`): on a status
  transition into `BudgetLimited` or `Complete`, increments the matching counter and emits
  `GOAL_TOKEN_COUNT_METRIC` and `GOAL_DURATION_SECONDS_METRIC` histograms tagged with
  `status`.
- `current_goal_status_for_metrics()` (`goals.rs:83-94`): reads the existing thread goal
  from `state_db` and matches the optional `expected_goal_id` before returning the
  status, so callers can capture the prior status before mutating.
- Three call sites: replace-goal path (line 17-29), new-goal path (line 33-37), and the
  accounting path (line 99-113) which captures `previous_status` *before*
  `account_thread_goal_usage` and emits afterwards.

## Assessment

The cardinality story is good — the only tag is `status`, which has 4 known values
(`Active`, `Paused`, `BudgetLimited`, `Complete`), and only 2 of those (`BudgetLimited`,
`Complete`) ever produce histogram emissions. That keeps Prometheus/OTel cardinality
bounded.

The "if status changed" guard at `goals.rs:56` is correct and necessary — without it the
accounting path would re-emit the terminal metric every poll once a goal hit its budget.
The `previous_status_for_goal` carve-out at lines 21-25 (when `replacing_goal`, treat
previous as `None` so the new goal's terminal status fires fresh) reads correctly.

Concerns:

- **`current_goal_status_for_metrics` race window** (line 83-94): reads from `state_db`
  to capture previous status, then `account_thread_goal_usage` mutates. If anything else
  on the same thread is writing the goal between those two calls, the captured
  `previous_status` is stale and the change-detection at line 56 will either miss a
  transition (false negative — we don't double-emit, fine) or spuriously detect one
  (false positive — we re-emit). Same-thread serialization probably makes this moot, but
  worth a comment confirming the invariant.
- **Histogram unit clarity**: `GOAL_DURATION_SECONDS_METRIC` is fed `goal.time_used_seconds`
  (line 78) — name and value agree. `GOAL_TOKEN_COUNT_METRIC` is fed `goal.tokens_used`
  with no unit suffix; conventionally OTel histogram metric names should encode units.
  Minor — depends on whether `codex_otel` already follows a convention.
- **No test coverage in this diff window**: the PR adds metric emissions but I don't see
  test asserts that a specific lifecycle (create → budget-limited) produces exactly N
  counter increments. Without that, future refactors can silently drop emissions. A
  unit test using a fake `session_telemetry` recorder would lock the contract.
- **The replace-goal path at line 26-29**: `if replacing_goal { self.emit_goal_created_metric(); }` —
  semantically a goal *replacement* is "old goal terminates, new goal begins." The new
  code emits the new-goal `GOAL_CREATED_METRIC`. But what about the *old* goal's terminal
  metric? If the replaced goal was `Active` at the time of replacement, the histogram for
  `goal.time_used_seconds` and `goal.tokens_used` is never emitted because the early
  return at line 63 skips `Active`/`Paused`. That data is lost. If the intent is "active
  goals that get replaced don't count as completed," that's defensible — but it should
  be documented. If the intent is "track every goal regardless of how it ended," you
  need a separate `Replaced` status or to emit unconditionally for replacement.

## Verdict

`merge-after-nits` — the architecture is right and the cardinality is bounded. Add (a) a
comment on the `previous_status` race-window invariant, (b) at least one unit test
asserting create→complete and create→budget-limited produce the expected metric
sequences, and (c) a note in the PR body on whether replaced-while-active goals are
intentionally lost from histograms. Not blocking on (a) and (c) — they're docs.
