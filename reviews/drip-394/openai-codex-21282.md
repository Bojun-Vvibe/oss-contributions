# Review: openai/codex #21282 — [codex] Remove legacy ListSkills op

- Head SHA: `1eab2471c0365e8e8b681c1f5289bd4d163c8651`
- Files: `codex-rs/core/src/lib.rs`, `codex-rs/core/src/session/handlers.rs` (and likely sibling protocol/event source removals not in visible window)

## Summary

Pure subtractive removal of the deprecated `Op::ListSkills` event/op surface.
The `list_skills` async handler in `session/handlers.rs` (≈ 90 lines) and
its supporting imports (`SkillError`, `SkillsLoadInput`,
`ListSkillsResponseEvent`, `SkillsListEntry`, `SkillErrorInfo`,
`load_config_layers_state`, `LOCAL_FS`, `AbsolutePathBuf`,
`CloudRequirementsLoader`, `LoaderOverrides`, `PathBuf`) all go with it.
The skills surface now lives entirely in app-server and the `SkillsWatcher`
push-notification path (per drip-388 PR review of the watcher migration).

## Specifics

- `codex-rs/core/src/lib.rs:91-94` — drops two `pub(crate) use skills::...`
  re-exports (`SkillError`, `SkillsLoadInput`). These were only used by the
  removed `list_skills` handler so the cleanup is consistent.
- `codex-rs/core/src/session/handlers.rs:18-58` — eight imports removed
  (`CloudRequirementsLoader`, `LoaderOverrides`, `load_config_layers_state`,
  `LOCAL_FS`, `AbsolutePathBuf`, `ListSkillsResponseEvent`,
  `SkillErrorInfo`, `SkillsListEntry`, `PathBuf`). All were exclusively
  consumed by `list_skills`; no orphaned `use` statements left behind.
- `codex-rs/core/src/session/handlers.rs:555-647` (visible window) — the
  full `pub async fn list_skills(...)` body (≈ 90 lines, including the
  per-cwd loop, the `AbsolutePathBuf::relative_to_current_dir` error
  branch, the `load_config_layers_state` error branch, the
  `effective_skill_roots_for_layer_stack` resolve, and the
  `skills_for_cwd(...)` call) is deleted in one block.

## Nits

- The PR is `[codex] Remove legacy ListSkills op` — confirm the corresponding
  `Op::ListSkills` variant in `codex-protocol` is also deleted in the same
  PR (not visible in the 300-line window). If it remains, out-of-tree
  consumers can still construct + send it and will silently get no reply,
  which is worse than a build break.
- No deprecation period / runtime no-op shim for the variant — if the public
  protocol-version contract requires a soft-removal cycle, this is a hard
  break. Worth a release-notes mention in the PR body if not already there.
- Sibling cleanup: confirm the `ListSkillsResponseEvent` type itself is
  dropped from `codex-protocol` — keeping a type whose only construction
  site was just deleted is a future-trap.

## Verdict

`merge-after-nits`
