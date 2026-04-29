# openai/codex #20175 — TUI: Remove core protocol dependency [4/7]

- **PR:** https://github.com/openai/codex/pull/20175
- **Head SHA:** 0a386c5d9d53b1e22e2bf61d00152734af3c807b
- **Files changed:** 81 files, +5936 / −8043
- **Verdict:** `merge-after-nits`

## What it does

The load-bearing 4-of-7 slice of the TUI ↔ core-protocol severance series. Swaps
TUI command, approval, realtime, skill, multi-agent, status, token-usage, diff,
history, and session-display paths over to `codex_app_server_protocol` types or
private TUI display models, and retires the legacy test event path. Net deletion
(−2107) despite touching 81 files because shared core-protocol shapes get replaced
with smaller TUI-owned models, and the pre-existing `app_server_adapter` is the
only surviving bridge (slated for deletion in 6/7 — see #20178).

## What's good

- The split into `app/app_server_event_targets.rs` (+218), `app/app_server_events.rs`
  (+186), and `app/thread_events.rs` (new file) keeps the per-concern dispatcher
  surface small and individually reviewable instead of a single `match` blob in
  `app.rs`.
- `app.rs:127-130` removing `use codex_models_manager::model_presets::HIDE_GPT_5_1...`
  shows the migration is genuinely scrubbing the imports rather than aliasing them.
- The `app_server_requests.rs:44-72` `ResolvedAppServerRequest` enum + `PendingApp
ServerRequests` struct gives the TUI its own request-correlation type instead of
  reusing the protocol crate's wire shape.
- Verification line in PR body is honest: `cargo test -p codex-tui --no-run` only —
  i.e. the author is asserting it builds and tests typecheck, not that they ran them.

## Nits / risks

1. `cargo test --no-run` is too weak a verification claim for a 81-file refactor that
   touches realtime + multi-agent + approval. The PR should state explicitly that the
   per-package suite passes (`cargo test -p codex-tui`) before merge — or the merger
   should run it.
2. The removal of `codex_protocol::protocol::McpStartupCompleteEvent` etc. (visible
   from #20172's diff base) means any external crate importing those test-only paths
   will fail. The boundary-enforcing `verify_tui_core_boundary.py` from the 7/7 slice
   should already be enabled in CI before this lands, otherwise drift between
   4/7 ↔ 7/7 reopens during the merge window.
3. Worth a one-line note in CHANGELOG that the TUI no longer depends on
   `codex_protocol` directly — downstream forks pinning a `codex_protocol` version
   will need to follow.

## What I learned

When a deletion-heavy refactor is split into a 7-PR stack, the load-bearing middle
slice (this one) is where the actual contract change lives; the 1-3/7 PRs prepare,
the 5-7/7 PRs delete the scaffolding. Reviewing 4/7 in isolation requires reading
the cover-letter for the whole stack, not just this PR.
