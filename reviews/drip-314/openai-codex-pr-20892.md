# Review — openai/codex #20892: feat(tui): add PR summary statusline items

- **PR:** https://github.com/openai/codex/pull/20892
- **Head SHA:** `075c29c4eac2bee96a1c57f955a74e5331f4a4b7`
- **Author:** fcoury-oai
- **Files:** 16 changed (+513 / −9)
  - `codex-rs/git-utils/src/info.rs` (+48) — new `branch_diff_stats_to_default_branch`
  - `codex-rs/core/src/git_info_tests.rs` (+58) — two new async tests
  - `codex-rs/tui/src/chatwidget/status_surfaces.rs` (+156/−5)
  - `codex-rs/tui/src/bottom_pane/status_line_setup.rs` (+26)
  - `codex-rs/tui/src/bottom_pane/chat_composer.rs` (+49)
  - + 11 other tui/onboarding files

## Verdict: `merge-after-nits`

## Rationale

Solid feature add. The new `branch_diff_stats_to_default_branch` in `git-utils/src/info.rs:277-316` is correctly written: it gets the repo root, resolves the default branch, computes `merge-base`, then diffs `merge_base..HEAD --numstat` and folds tab-separated columns into `(additions, deletions)`. The fall-through behaviour is the right call — every error path returns `None` so callers can render "—" instead of crashing the status line, and binary-file rows that emit `-\t-` become 0/0 via `parse().ok().unwrap_or(0)`. Tests in `git_info_tests.rs:434-489` cover both the clean-branch case and the committed-changes case. Renaming default to `main` first and then exercising the diff is correct given the test repo's initial branch name varies by host config.

Two things to address. First, the `branch_diff_stats_to_default_branch` numstat parser at `info.rs:300-309` silently coerces the binary-file marker `"-"` to `0` for both additions and deletions; that's defensible but the test coverage doesn't exercise it. A third test case adding a binary file (e.g. `fs::write(repo_path.join("img.bin"), &[0u8, 1, 2, 3])`) would document the intent. Second, `chat_composer.rs:193` adds `use crate::onboarding::mark_underlined_hyperlink;` — that import shows up in the diff but the diff hunks I reviewed don't show its call site (it's referenced further down). Verify the symbol actually gets used; an unused import will fail clippy.

The wiring through `AppEvent::StatusLineGitSummaryUpdated` (`app_event.rs:826-830`, dispatched in `app/event_dispatch.rs:1846-1850`) follows the existing async-update pattern for `set_status_line_branch`, which is good consistency. The git probe runs off the UI thread (per the surrounding `chatwidget.rs` patterns) so a slow `git diff --numstat` on a huge repo won't stall the TUI. Snapshot updates in `bottom_pane/snapshots/...status_line_setup__tests__setup_view_snapshot_uses_runtime_preview_values.snap` are a single-line delta which suggests a label string changed — verify the snapshot was reviewed by hand and not blindly accepted.

## Banned-string check

Diff scanned; no banned tokens present.
