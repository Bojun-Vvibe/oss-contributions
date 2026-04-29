---
pr: openai/codex#20246
sha: 8a6a8138ff539b2896feaaf1ba15564f8238627a
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# Gate multi-agent v2 tools independently of collab

URL: https://github.com/openai/codex/pull/20246
Files: `codex-rs/core/src/guardian/review_session.rs`,
`codex-rs/core/src/tasks/review.rs`,
`codex-rs/tools/src/tool_config.rs`,
`codex-rs/tools/src/tool_registry_plan_tests.rs`
Diff: 43+/1-

## Context

Multi-agent v2 tools (`spawn_agent`, `send_message`, `followup_task`,
`wait_agent`, `close_agent`, `list_agents`) were gated through
`include_collab_tools = features.enabled(Feature::Collab)` at
`tool_config.rs:145`. That meant turning on `MultiAgentV2` without
also enabling `Collab` silently delivered an empty toolset to the
model, and turning off `Collab` for a guardian/review sub-agent
unintentionally killed the multi-agent v2 surface too.

## What's good

- The fix at `tool_config.rs:145-147` is exactly the right shape:
  ```
  let include_multi_agent_v2 = features.enabled(Feature::MultiAgentV2);
  let include_collab_tools = include_multi_agent_v2 || features.enabled(Feature::Collab);
  ```
  reading as "collab-shaped tools ship if either flag is on", which
  matches the operational reality that multi-agent v2 is a strict
  superset of the collab tool surface.
- Sub-agent disable list at `tasks/review.rs:113` adds
  `MultiAgentV2` alongside the existing `SpawnCsv` and `Collab`
  disables — review sub-agents don't get to spawn further sub-agents,
  preserving the existing review-session containment invariant. The
  guardian config at `guardian/review_session.rs:882` mirrors the
  same explicit disable, so both code paths that build a
  containment-restricted feature set agree.
- New regression test
  `test_build_specs_multi_agent_v2_does_not_require_collab_feature`
  at `tool_registry_plan_tests.rs:386-422` is the right fence: it
  explicitly disables `Collab`, enables `MultiAgentV2`, asserts the
  six v2 tool names appear, and asserts the collab-only `send_input`
  / `resume_agent` *don't* appear (catching a future "v2 implies
  collab implies all collab tools" regression).
- The test asserts `assert!(!features.enabled(Feature::Collab))`
  after the disable+enable sequence — defensive, in case
  `features.disable()` has any "implied by another flag" semantics
  that would silently re-enable it.

## Nits

- Optional: a one-line comment at `tool_config.rs:145` naming the
  invariant ("multi-agent v2 is a superset of collab; turning on v2
  must imply collab tools without also enabling unrelated Collab
  features") would make the next refactor that adds a third gate
  obvious. Currently the relationship is implicit in the boolean
  `||`.
- The mirror test
  `test_build_specs_enable_fanout_enables_agent_jobs_and_collab_tools`
  at `:425+` exists for the inverse direction; consider adding one
  more permutation test "multi-agent v2 disabled, collab enabled →
  collab tools present, v2 tools absent" to fully fence the 2x2
  feature matrix.
- The two disable sites (`guardian/review_session.rs:882` and
  `tasks/review.rs:113`) are duplicated; long-term these would
  benefit from a shared `sub_agent_safety_features()` helper that
  returns the canonical disable set.

## Verdict reasoning

Minimal, correct fix to a real flag-coupling bug, with a regression
test that locks the new behaviour at exactly the right granularity.
Both sub-agent feature-set construction sites updated symmetrically.
Nits are doctrine/comment-only. Landable as-is.
