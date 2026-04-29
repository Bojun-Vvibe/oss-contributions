---
pr: openai/codex#20257
sha: bce1b3094522027ff63d35f23f9ad7e34779803c
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# Add remote execution environment to turn analytics

URL: https://github.com/openai/codex/pull/20257
Files: `codex-rs/analytics/src/analytics_client_tests.rs`,
`codex-rs/analytics/src/events.rs`, plus protocol/reducer wiring
Diff: 183+/2-

## Context

Codex turn analytics needs to distinguish locally-executed turns from
turns delegated to a remote execution environment so downstream
dashboards can split per-environment success/latency. PR threads a new
`Option<TurnExecutionEnvironment>` field through `TurnStartParams`,
`TurnSteerParams`, the `AnalyticsReducer`, and `CodexTurnEventParams`,
emitted as `event_params.execution_environment: "remote" | null` in the
serialized turn event.

## What's good

- Clean field addition at `events.rs:467-470` — `pub(crate)
  execution_environment: Option<TurnExecutionEnvironment>` slotted next
  to the existing thread provenance fields (`subagent_source`,
  `parent_thread_id`) where it belongs semantically.
- Two-call-site coverage in the existing test helper pattern:
  `sample_turn_start_request` and `sample_turn_steer_request` keep their
  current signature (delegate `None` through to the new
  `_with_execution_environment` variants) so existing tests don't churn.
- The serialization-shape lock in
  `turn_event_serializes_expected_shape` at `:1775,1837` adds the new
  key with explicit `"execution_environment": "remote"` — that's the
  right contract test for an analytics field where downstream
  dashboards depend on the exact JSON key name.
- New end-to-end test
  `turn_start_execution_environment_flows_to_turn_event` at `:2172-2243`
  exercises the full reducer pipeline (initialize → thread start →
  turn start with `Some(Remote)` → turn resolved config → turn
  completed) and asserts the field survives all four ingestion stages.
- Steer-path coverage at `:2261-2270` — the existing
  `accepted_steers_increment_turn_steer_count` test was extended to
  verify the field flows through steered turns too, not just initial
  starts. Catches the class of bug where turn-start sets the field but
  steer overwrites with `None`.

## Nits

- The new key is `Option<TurnExecutionEnvironment>` which serializes to
  `null` when absent. Downstream dashboards typically prefer a closed
  enum string over `null`; consider `#[serde(default = "default_local")]`
  emitting `"local"` so analytics queries can `WHERE
  execution_environment = 'local'` without `IS NULL` branching. Doc
  the decision either way.
- No test for the `None` (default) case asserting the serialized payload
  contains `"execution_environment": null` rather than the key being
  omitted entirely — `serde_json` defaults vary by Option/serde
  attribute combination and this matters for downstream BigQuery /
  Snowflake schema stability.
- The test extension at `accepted_steers_increment_turn_steer_count`
  now asserts both `execution_environment` and `steer_count` in one fn
  body — fine, but the test name no longer reflects what it's
  fencing. Either rename or split into a focused
  `steer_preserves_execution_environment` test.
- No CHANGELOG / event-schema-doc entry for downstream consumers. A new
  field is additive but consumers running strict schema validation
  (Looker, dbt models with `on_schema_change: fail`) need a heads-up.
- `TurnExecutionEnvironment` enum should ship with a doc comment
  enumerating the variants and their semantics (what does `Remote`
  mean? cloud sandbox? user-supplied SSH? Devbox?). Without that the
  analytics field is opaque to anyone reading the dashboard.

## Verdict reasoning

Correctly-shaped additive analytics-protocol change with thorough
end-to-end test coverage spanning both initial-start and steer paths.
The serialization-shape lock is in place. Nits are: (a) the `None →
null` vs `None → "local"` decision should be made deliberately not
incidentally, (b) downstream schema-doc entry, (c) doc comment on the
enum itself. None blocking; landable as-is with follow-up.
