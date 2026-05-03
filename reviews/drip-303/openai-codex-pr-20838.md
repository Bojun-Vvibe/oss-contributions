# openai/codex #20838 — Pause goals while in Plan mode

- URL: https://github.com/openai/codex/pull/20838
- Head SHA: `0c2f42979dc4efa36afc527e2e95a7dc18b2fc4b`
- Scope: +572 / -357 across 21 files
- Closes: #20656

## Summary

Resolves a state-model bug where `/goal` could appear active while Plan mode
prevented autonomous continuation, leading to confusing UX and broken
`/goal resume` flows. The fix moves Plan-mode handling out of the core goal
runtime and into the TUI/client layer: entering Plan mode now explicitly
pauses an active goal, and the user must explicitly resume it.

## What I checked

- `codex-rs/core/src/goals.rs` (-129) — significant simplification of core.
  Plan-mode-specific continuation suppression is gone. This is the right
  layering: core shouldn't know about UI modes.
- `codex-rs/app-server/src/codex_message_processor.rs:8467-8472` — removes the
  `apply_goal_resume_runtime_effects` call from the resume path. Combined with
  the new test
  `thread_resume_emits_paused_goal_snapshot_without_continuation`, this
  confirms paused goals stay paused on resume.
- `codex-rs/tui/src/app/thread_goal_actions.rs` (+53/-9) — the new TUI logic
  that sends `paused` when creating/resuming goals while in Plan mode. This
  is the new responsibility owner.
- `codex-rs/tui/src/chatwidget.rs` (+32/-6) and snapshots
  (`goal_menu_paused_plan_mode.snap`,
  `status_line_goal_active_plan_mode_footer.snap`) — UX coverage looks correct;
  the `/goal` view and footer both explain the paused state.
- Tests added across 5 layers: `app::tests` (+91), `chatwidget::tests::plan_mode`
  (+30), `chatwidget::tests::status_and_layout` (+46), `chatwidget::tests::goal_menu`
  (+18), and the app-server `thread_resume.rs` rewrite (+158/-154 — this is
  mostly a refactor to use shared `new_goals_enabled_mcp` / `start_test_thread`
  helpers, which is good but inflates the diff).
- The renamed test
  `thread_resume_emits_active_goal_update_before_continuation` →
  `thread_resume_emits_paused_goal_snapshot_without_continuation` accurately
  reflects the new contract (no continuation, just snapshot).

## Nits

1. The thread-resume test refactor (replacing 154 lines of inline setup with
   helpers) is the right cleanup but bundles two changes in one PR. A
   follow-up note in the commit message acknowledging this would help
   reviewers.
2. `goals.rs` loses 129 lines including what looks like a `Plan` enum branch.
   Confirm there are no remaining call sites in the workspace that still
   match on the removed variant — `cargo check -p codex-core` would catch it
   but a grep for `GoalContinuationMode` (or whatever the removed type is)
   across the repo would be reassuring.
3. The "explicit `/goal resume` still uses the normal active-goal continuation
   path" — would be nice to see an integration test that exercises:
   enter Plan mode → goal auto-pauses → exit Plan mode → user runs
   `/goal resume` → continuation actually resumes. The PR has unit-level
   coverage for each step but not the end-to-end happy path.
4. The TUI now owns Plan-mode goal pausing. If a non-TUI client (mcp-server,
   future web client) is added, it must replicate this logic. Worth a short
   note in the goal protocol docs that "pausing on Plan mode" is a client
   responsibility now.

## Risk

Medium. State-machine refactor across 5 crates (core, app-server, tui,
hooks-adjacent test files). Test coverage is solid (5 new test files
touched). The main risk is the layering shift — if a downstream client
relied on core auto-pausing goals in Plan mode, it now needs to do this
itself.

## Verdict

`merge-after-nits` — primarily the end-to-end resume integration test and
a docs note about the new client-layer responsibility.
