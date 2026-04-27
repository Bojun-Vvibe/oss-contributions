# openai/codex #19792 — multi_agent_v2: move thread cap into feature config

- **Repo**: openai/codex
- **PR**: #19792
- **Author**: jif-oai
- **Head SHA**: 9afa1a4400a25588e3f2c39198d09ca548b1f37e
- **Base**: main
- **Size**: +152 / −22 across config (`config/src/key_aliases.rs` −7,
  `core/config.schema.json` +5, `core/src/config/mod.rs` +34/−7,
  `core/src/config/config_tests.rs` +80, `core/src/session/turn_context.rs`
  +16/−2, `core/src/tools/handlers/agent_jobs.rs` +5,
  `features/src/feature_configs.rs` +3, `features/src/tests.rs` +3,
  `protocol/src/error.rs` ±1).

## What it changes

The PR migrates the session-thread cap that governs MultiAgentV2 from the
legacy `agents.max_threads` knob to a first-class
`features.multi_agent_v2.max_concurrent_threads_per_session` field, with
default `4` (root + 3 subagents). Three things move together:

1. The legacy alias mapping
   `[agents].max_concurrent_threads_per_session → max_threads` is dropped from
   `CONFIG_KEY_ALIASES` (`config/src/key_aliases.rs:11-22` deletion). Only the
   memories alias remains.
2. `MultiAgentV2Config` (`core/src/config/mod.rs:705-720`) gains a
   `max_concurrent_threads_per_session: usize` field with default
   `DEFAULT_MULTI_AGENT_V2_MAX_CONCURRENT_THREADS_PER_SESSION = 4`
   (`mod.rs:134`). Resolution order in `resolve_multi_agent_v2_config`
   (`mod.rs:1583-1610`) is profile → base → default, matching the existing
   `usage_hint_*` fields.
3. The translation to the legacy `agent_max_threads` (now meaning "max
   subagent threads, root excluded") happens in `Config::load` at
   `mod.rs:2087-2117`: when `Feature::MultiAgentV2` is enabled, setting
   `agents.max_threads` in TOML is rejected with
   `InvalidInput("agents.max_threads cannot be set when multi_agent_v2 is
   enabled")`, and `agent_max_threads` is derived as
   `multi_agent_v2.max_concurrent_threads_per_session.saturating_sub(1)`.
   When the feature is disabled, the previous `agents.max_threads` path
   (with its `Some(0)` rejection) is preserved verbatim.

`TurnContext::build_runner` and the per-turn rebuild in `Session`
(`session/turn_context.rs:194-202` and `:464-475`) gate the cap on the
feature flag with `features.enabled(MultiAgentV2).then_some(...)` instead of
unconditionally passing `config.agent_max_threads`. The agent-jobs handler
(`tools/handlers/agent_jobs.rs:534-541`) gets a new early-return when
`agent_max_threads == Some(0)` so a session with the cap set to 1 (root only)
returns `RespondToModel("agent thread limit reached; this session cannot
spawn more subagents")` instead of falling through into
`normalize_concurrency`.

## Strengths

- The "what does the cap count" semantics are now explicit and tested. The
  three new tests in `config_tests.rs` pin the contract:
  - `multi_agent_v2_default_session_thread_cap_counts_root` (`:7412-7430`):
    default cap `4` resolves to `agent_max_threads = Some(3)`, confirming
    root+3 subagent semantics.
  - `multi_agent_v2_rejects_agents_max_threads` (`:7432-7460`): coexistence
    with the legacy knob is a hard `InvalidInput` rather than a silent
    precedence rule, which is the right call — the previous alias quietly
    coerced one into the other and made the effective cap depend on which
    table the user wrote into.
  - `multi_agent_v2_session_thread_cap_one_disallows_subagents`
    (`:7462-7484`): cap `1` resolves to `Some(0)`, which combined with the
    new `agent_jobs.rs:537-540` short-circuit means the user gets a
    model-visible error (`agent thread limit reached…`) instead of a
    division-by-zero or silent zero-concurrency stall.
- Profile inheritance is preserved by the test
  `profile_multi_agent_v2_config_overrides_base` (`:7382-7407`): base sets
  `4`, profile sets `6`, resolved value is `6`. This matches the resolution
  helper at `mod.rs:1586-1591`.
- `saturating_sub(1)` at `mod.rs:2106-2110` is the correct primitive for
  cap=1 (becomes 0, not underflow). It also keeps the cap=0 path centralized
  in the new pre-check at `mod.rs:2090-2095` rather than scattering an
  off-by-one trap into the subtraction site.
- Removing the `max_threads` argument from
  `CodexErr::AgentLimitReached`'s `Display` (`protocol/src/error.rs:82-86`)
  makes sense: the cap value is no longer a single number from one config
  table, and surfacing it in the error string was already misleading once
  per-feature caps existed.

## Risks / nits

- `core/src/protocol/src/error.rs:85` keeps the `max_threads` field on the
  `AgentLimitReached` variant but drops it from `#[error(...)]`. That's
  fine for `Display`, but any downstream code matching on the variant and
  using `max_threads` to format its own message now silently loses meaning
  (the value is no longer "the cap" — it's `agent_max_threads`, which under
  MultiAgentV2 is `cap − 1`). Worth either renaming the field or adding a
  doc comment on the variant explaining that it's the subagent budget, not
  the session cap.
- The `agent_jobs.rs:537-540` short-circuit message uses
  `"agent thread limit reached; this session cannot spawn more subagents"`,
  which is duplicated in the depth-limit branch immediately above. Consider
  hoisting both into named constants so they stay in sync with whatever
  user-facing copy guidance applies to model-visible errors.
- The schema change at `core/config.schema.json:1310-1316` lists
  `max_concurrent_threads_per_session` with `format: uint, minimum: 1.0`
  but the runtime check at `mod.rs:2090-2095` already rejects 0. Good
  belt-and-braces, but note that the schema's `minimum: 1.0` is a float —
  most JSON-schema validators handle this fine, but some strict
  integer-only validators (e.g., the older `jsonschema` Python lib in
  draft-04 mode) treat `minimum: 1.0` differently from `minimum: 1`. Worth
  emitting an integer literal here.
- The deletion of the `agents → max_threads` alias means anyone who'd been
  setting `agents.max_concurrent_threads_per_session = N` and relying on it
  silently aliasing to `agents.max_threads` will now get the value ignored
  (it's not a known key) rather than a clear error. A one-release
  deprecation warning at parse time would be friendlier than the silent
  drop.
- `with_max_concurrent_threads_per_session(...
  .then_some(per_turn_config.multi_agent_v2.max_concurrent_threads_per_session))`
  at `session/turn_context.rs:467-475` reads a bit awkwardly — when the
  feature is disabled, it now passes `None`, but the previous code passed
  `config.agent_max_threads` (always `Some(_)`). If anything downstream
  treated `Some(_)` as "cap is enforced" vs `None` as "no cap", semantics
  shifted under feature-off. Worth a quick audit of consumers of
  `with_max_concurrent_threads_per_session` to confirm `None` still means
  "use the legacy default", not "disable enforcement".

## Suggestions

- Rename `CodexErr::AgentLimitReached.max_threads` to `subagent_budget` (or
  add a `#[deprecated = "use ..."]` alias) so the variant's payload reflects
  the new `cap − 1` semantics.
- Either add a one-release `agents.max_threads` migration warning emitted
  during config load when `MultiAgentV2` is enabled, or document the
  breaking change prominently in release notes — the rejection at
  `mod.rs:2095-2100` is the right end-state, but is a hard break for users
  whose configs have both tables.
- Consider asserting in the `multi_agent_v2_default_session_thread_cap_counts_root`
  test that the resolved cap shows up in the per-turn `TurnContext` (i.e.,
  end-to-end), not just in `config.multi_agent_v2`. The two values are now
  derived independently and a future refactor that forgets the
  `then_some(...)` gate would silently bypass enforcement.

## Verdict

`merge-after-nits` — the migration is well-scoped, the new tests pin the
exact semantics that the old alias-based path was hiding, and the
fail-loud rejection of `agents.max_threads` under MultiAgentV2 is the
right contract. Address the `AgentLimitReached.max_threads` field
semantics and add a deprecation note (or schema-emitted integer literal)
before merging.
