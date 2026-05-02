# openai/codex #20746 — Validate /goal objective length in TUI

- **Repo:** openai/codex
- **PR:** #20746
- **Head SHA:** `ec74cd2ae7e451acecedc09f18ab252dbe10a952`
- **Verdict:** merge-after-nits

## Summary

Adds TUI-side preflight validation for `/goal <objective>` against
`MAX_THREAD_GOAL_OBJECTIVE_CHARS`. Without it, oversized objectives
either hit a confusing lower-level failure or trip the generic
prompt-size error before the paste placeholder is expanded, hiding
the actual problem from the user.

New module `codex-rs/tui/src/chatwidget/goal_validation.rs`
(64 lines) introduces:

- `goal_objective_with_pending_pastes_is_allowed` — expands paste
  placeholders via `ChatComposer::expand_pending_pastes` then char-counts.
- `goal_objective_is_allowed` — used at queue-drain time.
- `goal_objective_char_count_is_allowed` — shared core. On failure it
  emits a goal-specific error referencing the file-redirect hint.

Wired into `slash_dispatch.rs:487-491` (live path) and
`slash_dispatch.rs:676-682` (queued path). Snapshot test in
`tests/goal_validation.rs` confirms the rendered error.

## Specific notes

- **Nit (`goal_validation.rs:39`):** `goal_objective_is_allowed` takes
  `&mut self` but the only mutation is `add_error_message` and
  composer reset (Live path only). Fine, just noting that the
  `&mut self` requirement leaks into the call site at
  `slash_dispatch.rs:679`. Not worth refactoring.
- **Nit (`goal_validation.rs:7`):** `GOAL_TOO_LONG_FILE_HINT` hardcodes
  the example path `docs/goal.md`. If you have an i18n / docs-style
  guide for these inline hints, consider whether `docs/goal.md` is
  too workspace-specific. Otherwise fine.
- **Live-path side effect:** when validation fails on Live source,
  the composer is cleared (`set_composer_text("", ...)` +
  `drain_pending_submission_state()`). That's a noticeable UX choice —
  the user loses what they just typed. Consider preserving the text
  so they can shrink it without retyping. Worth a follow-up.
- **Test coverage:** the snapshot at
  `snapshots/codex_tui__chatwidget__tests__goal_slash_command_oversized_objective_error.snap`
  pins the exact error text, which will catch any wording drift.
  Good.

## Rationale

Solid preflight that turns an opaque error into an actionable one.
The composer-clearing UX choice is the only thing I'd push back on,
but it's not a blocker.
