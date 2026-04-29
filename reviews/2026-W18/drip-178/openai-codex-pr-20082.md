# openai/codex#20082 — Use /goal resume for paused goals

- **Repo:** openai/codex
- **PR:** #20082
- **Head SHA:** 2fc275a215b643d715752b180fa59add09e7989f
- **Base:** main
- **Author:** maintainer

## Summary

Renames the slash-command verb from `/goal unpause` to `/goal resume` across the TUI. Six tiny edits: the doc comment on `AppEvent::SetThreadGoalStatus` at `codex-rs/tui/src/app_event.rs:222` (`unpause` → `resume`), the footer status hint at `codex-rs/tui/src/bottom_pane/footer.rs:551` (`"Goal paused (/goal to unpause)"` → `"Goal paused (/goal resume)"`), the goal-menu command-hint string at `codex-rs/tui/src/chatwidget/goal_menu.rs:48`, the dispatch `match` arm at `codex-rs/tui/src/chatwidget/slash_dispatch.rs:620` (`"unpause"` → `"resume"`), the existing snap file, and two test cases.

## File:line references

- `codex-rs/tui/src/app_event.rs:222` — doc comment update
- `codex-rs/tui/src/bottom_pane/footer.rs:551` — `GoalStatusIndicator::Paused` user-visible string
- `codex-rs/tui/src/chatwidget/goal_menu.rs:48` — `Commands: /goal resume, /goal clear`
- `codex-rs/tui/src/chatwidget/slash_dispatch.rs:620` — verb dispatch (only `"resume"` matches now; `"unpause"` is gone)
- `codex-rs/tui/src/chatwidget/snapshots/codex_tui__chatwidget__tests__goal_menu_paused.snap:11` — snapshot updated to match new string
- `codex-rs/tui/src/chatwidget/tests/slash_commands.rs:813` — test case renamed
- `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs:1845` — formatter test updated

## Verdict: **merge-after-nits**

## Rationale

Tiny UX-vocab change, well-snapshotted, low risk. Two real concerns:

1. **Hard removal of `"unpause"` is a silent breaking change for users with muscle memory.** `slash_dispatch.rs:620` no longer matches `"unpause"` at all, so users who type `/goal unpause` after upgrading get the generic "unknown command" path with no hint. Add a backward-compat arm:
   ```rust
   "resume" | "unpause" => Some(GoalControlCommand::SetStatus(AppThreadGoalStatus::Active)),
   ```
   and emit a deprecation hint to stderr / footer the first time someone uses `unpause` so the verb can be fully retired in a future release. If the maintainer prefers a clean rename, at minimum add a one-line CHANGELOG entry calling this out.
2. **No test asserts the "unknown verb" path for `/goal unpause` is graceful.** Add a one-line case to the table at `tests/slash_commands.rs:811-814` asserting `("/goal unpause", None)` produces the unknown-command path (or, if you adopt the alias above, that `unpause` still maps to `Active` for one release cycle).
3. The doc-comment change at `app_event.rs:222` is correct but the `SetThreadGoalStatus` enum variant itself isn't renamed — fine for source compatibility, but note it so future readers don't expect `Resume`/`Pause` variants symmetric to the verb.
4. Trivially: `bottom_pane/footer.rs:551` shortens the hint from "(/goal to unpause)" to "(/goal resume)" — losing the "to" word makes the parens read as a command name rather than a sentence. Consider `"Goal paused — type /goal resume"` for clarity (purely cosmetic).
