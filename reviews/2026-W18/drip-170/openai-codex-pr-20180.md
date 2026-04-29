# Review: openai/codex#20180 — Make multi-agent v2 ignore agents.max_depth

- **PR:** https://github.com/openai/codex/pull/20180
- **Head SHA:** `bd07d748728e22524b468539aa097adec55b42d2`
- **Author:** jif-oai
- **Diff size:** +56 / -7 across 4 files in `codex-rs/core/`
- **Verdict:** `merge-after-nits`

## What it does

Carves `agents.max_depth` out of the multi-agent v2 spawn path. The depth field is a v1 guard — v2 uses task-path routing and its own session/thread limits but still needs `depth` for lineage metadata. Three changes:

1. `codex-rs/core/src/agent/control.rs:521-524` — depth-based feature-disable now requires `MultiAgentV2` NOT enabled: adds `&& !config.features.enabled(Feature::MultiAgentV2)` to the existing `depth >= config.agent_max_depth` check.
2. `codex-rs/core/src/session/mod.rs:489-492` — the same guard applied at `Codex::start_session` (mirror site).
3. `codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs:45-50` — deletes the explicit `exceeds_thread_spawn_depth_limit(child_depth, max_depth)` rejection from the v2 spawn handler, plus the now-unused `use crate::agent::exceeds_thread_spawn_depth_limit` import in `multi_agents_v2.rs`.
4. `codex-rs/core/src/tools/handlers/multi_agents_tests.rs:1873-1925` — adds `multi_agent_v2_spawn_agent_ignores_configured_max_depth` test that sets `agent_max_depth = 1` + enables `Feature::MultiAgentV2`, then constructs a `SubAgentSource::ThreadSpawn { depth: 1, ... }` and asserts the v2 spawn returns success with `task_name == "/root/parent/child"`.

## Specific citations

- `agent/control.rs:521-525` and `session/mod.rs:489-493` are duplicate guards. The new condition is added consistently to both. Worth confirming with a follow-up grep that these are the only two sites that gate on `depth >= agent_max_depth`.
- `multi_agents_v2/spawn.rs:45-50` — the deleted block was the user-facing "Agent depth limit reached. Solve the task yourself." error string. After this PR, v2 callers will never see that string. If any user docs or eval prompts mention that exact error message, they need updating.
- `multi_agents_tests.rs:1873` — the new test is the discriminating one: it sets `agent_max_depth = 1` AND `depth = 1` (so v1 would have rejected) AND enables `MultiAgentV2`, then asserts success. Good test shape.
- The PR keeps `child_depth = next_thread_spawn_depth(&session_source)` at the top of the v2 spawn handler — so depth is still computed for the lineage path string, just not enforced.

## Nits to address before merge

1. **Add the negative test to round out the matrix.** The new test asserts "v2 enabled + depth at limit → success". The complementary case is "v2 enabled + depth WAY past limit (e.g. depth=100, max=1) → still success" to prove there's truly no upper bound being silently introduced elsewhere. Cheap to add.
2. **Document the rationale at the call sites.** The two guards in `control.rs:521` and `session/mod.rs:489` now read `depth >= max_depth && !v2_enabled`. The next contributor reading `agent_max_depth` will reasonably wonder "why doesn't this fire for v2?" — a one-line `// v2 uses task-path routing instead; see multi_agents_v2/spawn.rs` comment at each site prevents future "fix" PRs that put the guard back.
3. **The `agents.max_depth` config doc should be updated** to call out it is v1-only. If `codex-rs/core/docs/config.md` (or the README config table) lists this field, add "Ignored when `multi_agent_v2` feature is enabled."
4. **`exceeds_thread_spawn_depth_limit` is now dead code from the v2 path.** Confirm it still has v1 callers (a quick `rg exceeds_thread_spawn_depth_limit` in the diff context implied yes — `agent/control.rs` and `session/mod.rs` still gate on the inline `depth >= config.agent_max_depth` check, not via the helper). If the helper has zero callers post-merge, delete it; if it has only v1 callers, no action.
5. **Consider whether the v2 path should have ANY safety ceiling.** Even if `agent_max_depth` is "wrong shape", v2 likely wants a sanity cap (e.g. `1000`) to prevent runaway recursion from a misbehaving model, surfaced as a different error string. Out of scope for this PR but worth filing as a follow-up issue.

## Rationale

Surgical, well-tested change that correctly inverts a v1-only guard for v2 callers. The diff is minimal, the test is discriminating (sets both `max_depth` and `depth` to the same value to prove the guard would have fired in v1), and the two mirror sites in `control.rs` and `session/mod.rs` are kept consistent. Gaps are commenting/docs and one paranoia test. None block merge.