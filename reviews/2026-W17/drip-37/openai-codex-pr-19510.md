# openai/codex #19510 — Hide rewind preview when no user message exists

- **Repo**: openai/codex
- **PR**: [#19510](https://github.com/openai/codex/pull/19510)
- **Head SHA**: `aea304564500bfefe77a357495c823bcbe99738c`
- **Author**: etraut-openai
- **State**: OPEN (+81 / -2)
- **Verdict**: `merge-as-is`

## Context

In a fresh TUI session, double-`Esc` would open the rewind transcript
overlay even when there were zero user messages to rewind to —
producing an empty header-only view and a state machine sitting on
a "select a target" step that has no valid targets. Issue #19508.

## Design

Three carve-outs around a single new predicate, plus the matching
hint suppression:

1. `has_backtrack_target(cells)` is `user_count(cells) > 0`
   (`codex-rs/tui/src/app_backtrack.rs:637-639`). Cheap, reuses the
   existing iterator. Lives next to `user_count` and the other
   target-resolution helpers — right place for it.

2. `App::open_backtrack_preview` (lines 277-289) and
   `App::begin_overlay_backtrack_preview` (lines 296-309) both
   check `has_backtrack_target` before priming. Miss → reset
   backtrack state, push a `NO_PREVIOUS_MESSAGE_TO_EDIT` info
   message, request a frame, return. The overlay variant also
   closes the overlay first since it was opened by the user's
   `Ctrl+T` before this entry point fired. Both branches go through
   `add_info_message` rather than logging or silently no-op'ing,
   which is the right user-visible signal.

3. `prime_backtrack` (line 269-273) gates the
   `show_esc_backtrack_hint` call on the same predicate, so the
   composer footer doesn't lie about an "edit previous message"
   shortcut when there's nothing to edit.

Tests:

- `backtrack_target_requires_user_message` (lines 928-953) covers
  the assistant-only and assistant + info-event cases (target
  absent), then pushes a `UserHistoryCell` and re-asserts
  (target present). Good boundary coverage on the predicate.
- `backtrack_unavailable_info_message_snapshot` (lines 955-963)
  pins the literal string + render shape via insta — catches
  accidental wording drift.
- The new snapshot file
  `codex_tui__app_backtrack__tests__backtrack_unavailable_info_message_snapshot.snap`
  contains exactly `• No previous message to edit.` — matches the
  bullet-rendered info-event style used elsewhere.

## Risks

Very low. The reset on the no-target overlay path
(`reset_backtrack_state` in `begin_overlay_backtrack_preview`)
matters: without it, `primed=true` and `base_id` could leak into
the next `Esc` press. The PR sets `primed=true` in the *non*-empty
branch only, which is correct — but worth a sanity check that
existing reset paths cover the case where the overlay was opened
manually via `Ctrl+T` then dismissed. From reading
`close_transcript_overlay` and the existing
`reset_backtrack_state`, this looks fine.

## Suggestions

None blocking. One nit: the constant
`NO_PREVIOUS_MESSAGE_TO_EDIT` is defined at module scope but only
used in two call sites and one snapshot — that's the right shape
because the snapshot test directly references it, so any future
wording change has a single source of truth and the snapshot will
fail loudly.

## What I learned

The failure mode here — UI surfaces a flow whose state machine has
no valid initial transition — is a classic "the entry point and
the precondition live in different files" bug. The fix is the
right shape: lift the predicate next to the state, gate at every
entry point, and surface a user-visible info message rather than
silently swallowing the keypress (which would have left users
wondering whether `Esc Esc` is broken).
