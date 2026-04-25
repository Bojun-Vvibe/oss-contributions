# codex #19591 — Fix TUI resume performance regression

- URL: https://github.com/openai/codex/pull/19591
- Head SHA: `319d4e1f9cb6b4e8ca69d96cc260636c7ff9421e`
- Verdict: **merge-as-is**

## What it does

After PR #18502 added multi-CWD `thread/list` filtering, the default code
path went back to JSONL scan-and-repair unless callers explicitly opted into
the SQLite-only fast path via `use_state_db_only: true`. The TUI resume
flow only needs the materialized thread stats, so `codex resume` regressed
into walking JSONL transcripts on every invocation — visibly slow on users
with long histories.

This PR flips three TUI call sites to `use_state_db_only: true`:

- `codex-rs/tui/src/lib.rs:528` — `lookup_session_target_by_name_with_app_server`
- `codex-rs/tui/src/lib.rs:621` — `latest_session_lookup_params`
- `codex-rs/tui/src/resume_picker.rs:1007` — `thread_list_params`

…and pins the new behavior in three existing tests with extra
`assert!(params.use_state_db_only);` checks.

## Diff notes

- The change is purely in TUI call sites, not in the underlying
  `app-server` or `core` query code, so non-TUI consumers (CI scripts,
  MCP integrations) keep the safer scan-and-repair default.
- The picked path is correct: TUI resume specifically wants "what does the
  database currently know about my recent threads", not "rebuild thread
  index from on-disk JSONL". Anything missing from SQLite is also missing
  from the picker UI in any sensible interpretation.
- Test additions are minimal but exactly what's needed — they assert the
  field is `true` in the params struct, which is the contract layer that
  the next refactor would most likely violate.

## Risk surface

- Edge case: a user whose SQLite state DB is out of sync with their JSONL
  files (e.g., they restored only `.jsonl` from a backup) will see fewer
  threads in the picker than before. That's acceptable — the fix for that
  is to re-run whatever rebuilds the state DB, not to make every resume
  walk JSONL.
- No change to write-side behavior, no schema migration. Worst case is "I
  see fewer entries", never "I corrupt history".

## Why this verdict

Surgical regression fix with the right scope (TUI only), the right tests
(field-level pins on the params struct), and a clear written rationale
tied back to the offending PR. Nothing to nit.
