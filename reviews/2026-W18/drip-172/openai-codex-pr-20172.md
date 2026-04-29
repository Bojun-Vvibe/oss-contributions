# openai/codex #20172 — TUI: Remove core protocol dependency [1/7]

- **PR:** https://github.com/openai/codex/pull/20172
- **Head SHA:** aa4be6efcc97fef987fce27df4d0d9c390490588
- **Files changed:** 4 files, +269 / −253
- **Verdict:** `merge-as-is`

## What it does

Slice 1 of the TUI ↔ core-protocol severance series. This is a pure module extraction:
the ~250-line MCP-startup state machine living inside `codex-rs/tui/src/chatwidget.rs`
(`update_mcp_startup_status`, `set_mcp_startup_expected_servers`,
`finish_mcp_startup`, `finish_mcp_startup_after_lag`, `on_mcp_server_status_updated`,
plus the test-only `on_mcp_startup_update` / `on_mcp_startup_complete`) gets lifted
into a new `codex-rs/tui/src/chatwidget/mcp_startup.rs` module (+265). The
`mod mcp_startup;` declaration lands at `chatwidget.rs:389`, and the corresponding
`use std::collections::BTreeSet;` / `McpServerStartupState` / `McpStartupCompleteEvent`
/ `McpStartupUpdateEvent` imports are removed from `chatwidget.rs:33,98,191,194` —
all moved into the new module instead, which is the right cleanup.

## What's good

- Pure refactor: net ~+15 / −15 in behavior-bearing code. The +269 / −253 totals come
  from the move itself, not from logic edits. A diff-of-diffs (or `git log -p
  --follow`) on the new file vs the old block should show no semantic change.
- Carving out `mcp_startup.rs` first means the boundary-enforcing changes in 4/7
  (#20175) and the deletion in 6/7 (#20178) only have to delete one import path
  instead of fighting `chatwidget.rs` line-by-line.
- The doc comment on `update_mcp_startup_status` (the "ignore-mode buffers the next
  plausible round" lossy-delivery rationale) survives the move intact (visible at
  `mcp_startup.rs` ~line 13 onward) — that comment is the actual contract for
  `mcp_startup_ignore_updates_until_next_start` / `..._pending_next_round_saw_starting`
  and would have been very expensive to re-derive if it had been dropped.

## Nits

None blocking. Optional: a one-line `// extracted from chatwidget.rs in PR #20172`
header at the top of the new file would help future archaeology, but the git history
is enough.

## Verification suggestion

`cargo test -p codex-tui chatwidget::mcp_startup` should still pass with zero
behavior-test changes; if any test name moved, that's the audit signal.

## What I learned

A 7-PR severance stack that opens with a pure code-move (1/7) instead of a behavior
change is the right shape — the load-bearing semantic edits in 4/7 then have a
narrower diff to review and nothing accidentally rides along.
