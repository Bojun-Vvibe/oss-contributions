# openai/codex #19506 — [codex] Refresh AGENTS.md on cwd changes

- **Repo**: openai/codex
- **PR**: [#19506](https://github.com/openai/codex/pull/19506)
- **Head SHA**: `3ce2525848ff4d8a921cac7271a3c91a4a2a8414`
- **Author**: Aismit
- **State**: OPEN (+51 / -21)
- **Verdict**: `merge-after-nits`

## Context

`AGENTS.md` (project doc instructions) is currently resolved once at
session start and frozen for the whole thread. When the agent issues
`cd` (or any tool that mutates the effective turn cwd), subsequent
turns continue to see the *original* directory's instructions — even
though the model is now operating in a different project. This PR
refreshes the AGENTS.md fragment per turn against the effective cwd
and emits a model-visible user-instructions update only when the
resolved content actually changes.

## Design

Two parallel changes:

1. **Per-turn resolution**, in
   `codex-rs/core/src/session/turn_context.rs:626-689` — the new
   `AgentsMdManager::new(&per_turn_config).user_instructions(environment.as_deref()).await`
   call resolves AGENTS.md against the effective per-turn config, and
   the result is written into `turn_context.user_instructions` at line
   689. This is the right layering: the manager already exists, it
   just was never being asked again after session boot.

2. **Diff-gated emission** in
   `codex-rs/core/src/context_manager/updates.rs:215-251`. The
   settings-update path now compares `previous.user_instructions` vs
   `next.user_instructions` and only pushes a `UserInstructions`
   ContextualUserFragment when they differ. Critically, when emitted
   it carries `directory: next.cwd.to_string_lossy().into_owned()` so
   the model sees both the new text *and* the cwd it was resolved
   from. This avoids the failure mode where the model sees fresh
   instructions but no signal about which project they apply to.

The new snapshot test
`snapshot_model_visible_layout_cwd_change_refreshes_agents` at
`tests/suite/model_visible_layout.rs:181-200` asserts that turn-2
contains *both* the original and refreshed instruction blocks (lines
113-128) — i.e. the refresh is additive in the visible context, not a
silent swap. The complementary test for the no-change case
(`...does_not_refresh_agents`) already existed and now passes by
construction because the diff-gate suppresses the emit.

## Risks

- **Latency**: `AgentsMdManager::new(...).user_instructions().await`
  runs every turn, including I/O. If a user has a slow filesystem or
  a deeply nested AGENTS.md hierarchy, every turn now eats that
  cost. Worth adding a stat-based fast path or a directory-keyed
  cache before merge.
- **Equality check**: `previous.and_then(|prev| prev.user_instructions.as_ref()) == next.user_instructions.as_ref()`
  at `updates.rs:217` compares `Option<&Arc<String>>` (or similar). If
  the AGENTS.md content is identical but the `Arc` pointer differs,
  this is a false-negative — the gate fires and re-emits. Should be
  a content-equality, not an `Option<&_>` PartialEq derived match.
- **Token cost**: turn-2 now carries *two* AGENTS.md blocks
  (verified by the snapshot). Over many cd hops this grows
  unboundedly. Consider a cap or a "supersedes prior" semantic.

## Verdict

`merge-after-nits` — fix the equality check and consider the cache
before landing. The behavior change is correct and the snapshot test
locks it in.
