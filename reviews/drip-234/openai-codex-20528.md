# openai/codex#20528 — support skill-scoped hooks

- **PR**: https://github.com/openai/codex/pull/20528
- **Head SHA**: `0615194a36758dd91872f391558eb910786a1029`
- **Size**: +426 / -18, 24 files
- **Verdict**: **merge-after-nits**

## Context

Codex Skills (markdown files with frontmatter, loaded from system/user/project scopes) gain a new `hooks:` frontmatter block, and those hooks become *active only when the skill is mentioned in the current turn*. This is a real behavioral expansion: previously hooks lived in `config.toml`-tier `Hooks` and were globally on; now skills can ship their own `PreToolUse` / `PostToolUse` / `PermissionRequest` / `Stop` hooks scoped to skill activation.

## Design analysis

### Frontmatter & loader (`core-skills/src/loader.rs`)

`SkillFrontmatter` at `:42-46` gains `#[serde(default)] hooks: HookEventsToml` — same schema as `config.toml` hooks, so users get one mental model. `parse_skill_file` at `:592-660` now returns a private `ParsedSkill { metadata, hooks }` struct, the loader at `:557-565` writes `(path_to_skills_md → hooks)` into `outcome.hooks_by_skill_path` then pushes the metadata. Type signature change from `Result<SkillMetadata>` → `Result<ParsedSkill>` is internal.

### Outcome model (`core-skills/src/model.rs`)

`SkillLoadOutcome` at `:95` adds `pub(crate) hooks_by_skill_path: HashMap<AbsolutePathBuf, HookEventsToml>`. New accessor `hooks_for_skill(&SkillMetadata) -> Option<&HookEventsToml>` at `:131-133`. `filter_skill_load_outcome_for_product(...)` at `:184-187` correctly retains the same paths in the new map as it does for the existing `file_systems_by_skill_path`. Matched-pair retention is the right pattern — if a future filter is added, both maps go through the same retain set.

### Hook-source plumbing (`core/src/skills.rs:233-272`)

New `active_skill_hook_sources(outcome, &mentioned_skills)` returns `(Vec<SkillHookSource>, Vec<String>)`. For each mentioned skill: `outcome.hooks_for_skill(skill)` → if non-empty, clone the hooks, **strip `session_start` with a warning** ("Skipping SessionStart hooks from skill X because skills activate after thread start"), then push a `SkillHookSource { skill_name, source_path, hooks }`.

The session-start carve-out is load-bearing and correct: skill activation happens *after* the thread starts, so `SessionStart` hooks from a skill could never fire deterministically — the warning makes that visible to the user instead of silent drop.

### Turn-scoped hook merging (`core/src/session/turn_context.rs:381-419`)

`TurnSkillsContext` gains `active_hook_sources: Arc<RwLock<Vec<SkillHookSource>>>` plus:
- `set_active_hook_sources(sources)` — called once per turn after mention detection.
- `hooks_for_turn(base_hooks: Arc<Hooks>) -> Arc<Hooks>` — returns `base_hooks` unchanged if no skill hooks, else `Arc::new(base_hooks.with_skill_hook_sources(active_hook_sources))`.

The `if active_hook_sources.is_empty() { base_hooks } else { ... }` short-circuit is the load-bearing zero-cost path: turns without any active skill hooks pay no allocation. Excellent.

### Call sites (`core/src/hook_runtime.rs`)

Four call sites flipped consistently from `let hooks = sess.hooks();` → `let hooks = turn_context.turn_skills.hooks_for_turn(sess.hooks());`:
- `run_pending_session_start_hooks` `:119` — note: skill `session_start` already stripped at source, so this only sees config-tier session-start hooks. Defensive.
- `run_pre_tool_use_hooks` `:156`
- `run_permission_request_hooks` `:206`
- `run_post_tool_use_hooks` `:247`

Plus the stop hook in `core/src/session/turn.rs:537`. Five call sites — exhaustive coverage of the existing hook surface, no path forgotten.

### Feature gating (`core/src/session/turn.rs:223-235`)

```rust
if turn_context.features.enabled(Feature::CodexHooks)
    && let Some(outcome) = skills_outcome
{
    let (skill_hook_sources, skill_hook_warnings) =
        active_skill_hook_sources(outcome, &mentioned_skills);
    turn_context.turn_skills.set_active_hook_sources(skill_hook_sources);
    for message in skill_hook_warnings {
        sess.send_event(&turn_context, EventMsg::Warning(WarningEvent { message })).await;
    }
}
```

Gated on the existing `CodexHooks` feature flag — if hooks are off globally, skill hooks are also off. Symmetric with config-tier hooks. Warnings emitted to the user as `WarningEvent` — visible.

### Wire protocol (`HookSource` enum, 8 schema files)

`HookSource::Skill` added to:
- `app-server-protocol/src/protocol/v2.rs:475`
- 6 generated JSON schemas (`ServerNotification.json`, `codex_app_server_protocol.schemas.json`, `..v2.schemas.json`, `v2/HookCompletedNotification.json`, `v2/HookStartedNotification.json`, `v2/HooksListResponse.json`)
- `schema/typescript/v2/HookSource.ts:5` — added between `plugin` and `cloudRequirements`, preserving enum order
- `analytics/src/events.rs:688` and `core/src/hook_runtime.rs:481` — both string-mapping sites flipped consistently to `"skill"`

Schema-update discipline is exemplary — all 6 JSON files + the TS type + both Rust string-map sites updated atomically. Same discipline as #20530 multi-env, #20560 plugin-share. This is the kind of change that goes wrong silently if any consumer is missed; the matched-set update prevents that.

### Test (`core/tests/suite/hooks.rs:1976-2141`)

New `skill_pre_tool_use_blocks_only_while_skill_is_active` test (referenced; visible in diff `:1976-2141` but truncated). Pins the contract: PreToolUse hook from a skill blocks the matching tool call only on turns where the skill is mentioned, and a subsequent turn without the mention executes the same tool unblocked.

Plus loader test at `loader_tests.rs:328-373` (`parses_hooks_from_skill_frontmatter`) pinning that frontmatter `hooks: PreToolUse: [{matcher: "Bash", hooks: [{type: command, command: "..."}]}]` round-trips through the loader and is retrievable via `outcome.hooks_for_skill(skill)`.

## What's right

- **Mention-scoped activation** is the load-bearing UX choice. Global hooks-from-skills would make every skill an always-on hook, defeating the per-turn "skill is contextually relevant" model.
- **`SessionStart` strip-with-warning** rather than silent drop or hard error — exactly the right call for a class of hook that *physically cannot fire* given when skills activate.
- **`active_hook_sources.is_empty() → base_hooks` short-circuit** keeps turns with no skill-hooks at zero cost (no extra `Arc::new`, no merge).
- **Schema/string-map exhaustive update** across 8+ files for the new enum variant.
- **Two test arms** at the contract boundary (loader parse + runtime activation gate).

## Nits / discussion

- **`PoisonError::into_inner` on `unwrap_or_else`** in both `set_active_hook_sources` and `hooks_for_turn`. This is the right pattern (don't propagate poison), but a brief comment explaining the policy ("hook context is per-turn ephemeral, recovering from poison is safe") would help the next maintainer.
- **`sync::RwLock` inside `Arc`** when the consumers are async (the call sites are in `async fn`s). Using `tokio::sync::RwLock` would let the read path `.await` without blocking; right now `read().unwrap()` is sync-blocking. Given the lock is held for a clone-and-drop on a `Vec<SkillHookSource>` that's typically short, the contention is minimal — but if a turn has many tools and each tool triggers a hook lookup, this is repeated sync acquisition. Worth a benchmark if a profile flags it.
- **`active_skill_hook_sources` consumes `&mentioned_skills`** which is `Vec<SkillMetadata>` (clones from the iterator). For turns mentioning many skills, repeated `outcome.hooks_for_skill(skill).clone()` on the `HookEventsToml` could be measurable. Not a hot path today, fine for v1.
- **`Skill` enum variant placement** between `Plugin` and `CloudRequirements` — fine for the JSON schemas (string-tagged) but for any binary-protocol consumer that does `match` ordering, this could change ordinals. Codex AFAICT only serializes via tag-string, so safe.
- **No mention of analytics surface for skill-hook firing rates.** Once skill hooks ship, we'll want to know "how often does a skill's hook actually fire vs. configured" — a follow-up `SkillHookFired` analytics event keyed by `skill_name + hook_kind` would close that loop.
- **Migration story for users with existing `config.toml` hooks**: not needed — additive — but worth a one-line in the PR body.

## Verdict

**merge-after-nits** — well-architected feature with the right scope discipline (mention-gated activation, session-start strip-with-warning, zero-cost no-skill-hook path, exhaustive schema/string-map updates). The nits are documentation/benchmark items, not correctness blockers.
