# openai/codex #20913 — frodex: restore TUI subagent surface

- SHA: `efdf88acb610e50345d4d3614597cffdd047b4de`
- State: OPEN, +16459/-1201 across ~44 files
- Stack: base `dev/friel/frodex-129-watchdog-parent-messaging`, next `dev/friel/frodex-129-fork-command-debug`
- Files of interest: `codex-rs/tui/src/subagent_panel.rs` (+235), `codex-rs/tui/src/chatwidget.rs` (+185/-24), `codex-rs/tui/src/history_cell.rs` (+229), 9 new snapshot tests under `chatwidget/snapshots/`, `chatwidget/tests/app_server.rs` (+403), `app-server-protocol` schema regen (+~9000 lines, mostly mechanical).

## Summary

Restores the live subagent panel in the TUI: watchdog status rendering, `/agent` filtering for watchdog/closed-named rows, subagent completion cells, fork-parent title display, watchdog snooze + close cells. Most of the additions are JSON-schema regen for `app-server-protocol` (`ContentItem`, `FunctionCallOutputBody`, `LocalShellAction`, `ImageDetail` definitions added across all v2 thread/turn responses). Real Rust code changes are concentrated in `tui/src/`.

## Notes

- `codex-rs/tui/src/subagent_panel.rs:1-235` — new module; needs a quick read for boundary cases (empty roster, watchdog without parent thread, closed-named rows being re-resumed). I'd ask for a test that covers a watchdog row whose parent has been archived mid-session.
- `codex-rs/tui/src/chatwidget.rs:24-209` — most of the +185 lines are panel-mount / row-resurrect plumbing. The 6 new snapshot tests (`subagent_panel_mounts_watchdog_spawn`, `subagent_panel_renders_subagent_and_watchdog_rows`, `watchdog_goodbye_message_closes_subagent_panel_row`, `watchdog_goodbye_message_inserts_close_history`, `resume_replay_open_subagent_history_cells`, `resume_replay_does_not_resurrect_open_subagent_panel_row`, `resume_replay_closed_watchdog_history_cells`, `resume_replay_does_not_resurrect_closed_watchdog_panel_row`, `live_app_server_raw_inter_agent_message_renders_agent_message_cell`) cover the resume-replay no-resurrect invariants well.
- `codex-rs/app-server-protocol/schema/json/**` — +~9000 lines of regen. These are autogen artifacts; would be much easier to review if the schema regen were split into a separate "chore: regen schema" commit. Currently they bloat the diff and obscure the actual logic changes.
- `codex-rs/app-server/tests/suite/v2/thread_resume.rs` (+151) and `chatwidget/tests/app_server.rs` (+403) provide end-to-end coverage.
- `core/src/tools/handlers/multi_agents/send_input.rs` (+48/-4) and `multi_agents_tests.rs` (+57) — small but real semantic change to send_input. Need to verify it doesn't regress non-subagent send_input flows.
- `tui/src/app/agent_navigation.rs` (+1/-15) — net negative; looks like dead code removal as part of the panel rewrite. OK.
- PR body notes one upstream Unix socket IPC test fails on the devbox with a tempdir permission error — should be reproduced or explicitly waived in CI.
- This is part of a stack (base + next branches named); reviewer needs the base branch context to judge correctness.

## Verdict

`needs-discussion` — the substantive Rust changes look reasonable but the +16k diff is dominated by schema regen that should be in a separate commit, and the panel-mount / resume-replay logic is intricate enough to warrant a stacked-PR walkthrough. Cannot in good conscience approve a 16k-line diff without splitting; ask author to factor schema regen out, then re-request review.
