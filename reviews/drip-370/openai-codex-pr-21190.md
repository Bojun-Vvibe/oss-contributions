# openai/codex PR #21190 — fix(tui): external editor expansion for same-size large pastes

- URL: https://github.com/openai/codex/pull/21190
- Head SHA: `f868febdbe32dccf3715468f7084371d14f7df1c`
- Author: fcoury-oai
- Size: +33 / -5 (one file: `codex-rs/tui/src/bottom_pane/chat_composer.rs`)

## Verdict
`merge-as-is`

## Rationale

This is a tight correctness fix for a real user-visible corruption bug. The pre-existing
`current_text_with_pending()` at `chat_composer.rs:1120-1126` did global string `text.replace(placeholder,
actual)` in a loop over `pending_pastes`. When two same-size pastes are active, their placeholders are
prefix-overlapping — `[Pasted Content 1004 chars]` and `[Pasted Content 1004 chars] #2` — so replacing
the base placeholder first rewrites the prefix of the `#2` placeholder, leaving the external editor
seeded with `<first payload> #2` instead of `<first payload><second payload>`. Same-length is the
worst case because the dedupe-by-suffix scheme assumes payload length differentiates labels, which
it doesn't here.

The fix routes the call through the existing `Self::expand_pending_pastes(&text,
self.current_text_elements(), &self.pending_pastes)` helper (the same path the other materialization
sites already use), so global string replacement is dropped in favor of element-range expansion that
walks `current_text_elements()` once and substitutes by token identity rather than text. The
`is_empty()` early return at `:1124` preserves the no-allocation fast path. The regression test at
`:10138-10164` is well-targeted: it constructs the exact overlap scenario (`first_paste = "a".repeat(LARGE_PASTE_CHAR_THRESHOLD + 4)` and `second_paste = "b".repeat(...)`,
both ≥ threshold so both materialize as pending), pins the visible composer text as the literal
`{base}{second}` concatenation, and asserts `current_text_with_pending()` produces
`{first_paste}{second_paste}` (i.e. both fully expanded with no `#2` literal left behind).

Scope is correct — author explicitly notes #21091 fixes the *numbering-after-clear* path while this
PR fixes the *materialization-with-overlap* path, and they're orthogonal. No public API change, no
test deletions, the existing `expand_pending_pastes` helper is untouched. The `cargo test -p
codex-tui` two unrelated failures are clearly local-machine env (`/etc/codex/requirements.toml`
rejects `DangerFullAccess`) and have a prior history with the test harness, not this PR. Ready to
land.
