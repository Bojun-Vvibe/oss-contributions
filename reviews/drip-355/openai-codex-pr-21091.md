# openai/codex PR #21091 — Fix TUI large-paste placeholder numbering after Ctrl+C

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/21091
- Head SHA: `473cff8776d5`
- Size: +97 / -10 across 1 file (`codex-rs/tui/src/bottom_pane/chat_composer.rs`)
- Fixes upstream #19940.

## Summary

The TUI inserts `[Pasted Content N chars]` placeholders for large pastes,
with a `#2`/`#3`/... suffix when multiple same-size pastes co-exist in
the buffer. The numbering was backed by a per-size counter
(`large_paste_counters: HashMap<usize, usize>`) on the composer struct.
That counter was never reset when the user cleared a draft with Ctrl+C —
so after clearing, the *next* paste of the same size would show up as
`[Pasted Content 5000 chars] #5` even though no other pastes of that size
were live anywhere. This PR drops the counter entirely and derives the
suffix from the placeholders that still exist in `pending_pastes`,
making pending-paste state the single source of truth.

## What I like

- The fix is a deletion: `large_paste_counters: HashMap<usize, usize>` is
  removed from the struct (chat_composer.rs:341, line removed) and from
  `new()` (chat_composer.rs:539, line removed). One less piece of mutable
  state to keep in sync with anything else.
- `next_large_paste_placeholder` becomes `&self` instead of `&mut self`
  (chat_composer.rs:1637). A pure read of `pending_pastes` is the entire
  computation — that change alone is a strong signal the new design is
  right.
- Suffix derivation at chat_composer.rs:1639-1654 walks
  `self.pending_pastes`, treats a bare `[Pasted Content N chars]` (no
  suffix) as suffix=1, and picks `max_suffix.max(parsed_suffix)` for any
  `[Pasted Content N chars] #K`. Then returns base for `max_suffix==0`
  and `format!("{base} #{}", max_suffix + 1)` otherwise. This correctly
  handles "user deleted #2 but kept #1 and #3 → next is #4" without
  needing a separate "max ever seen" counter.
- The new test
  `large_paste_numbering_reuses_after_ctrl_c_clear` at
  chat_composer.rs:5743+ exercises the exact failure mode: paste, clear
  with Ctrl+C, paste again, assert the second placeholder is the bare
  `[Pasted Content N chars]` with no `#2` suffix.
- The expanded module doc comment at chat_composer.rs:60-69 documents
  the new contract clearly: "When all placeholders for a size are
  cleared or deleted, the next paste of that size reuses the base label
  without a suffix." Matches the implementation.

## Nits / discussion

1. **Linear scan on every paste.** The new derivation iterates all
   `pending_pastes` entries on each large paste
   (chat_composer.rs:1641-1652). For typical sessions `pending_pastes`
   is short (≤10), so this is a non-issue. But if anyone ever lifts the
   placeholder limit, a `HashMap<usize, BTreeSet<usize>>` keyed by
   char_count would scale better. Not worth blocking on.

2. **Suffix-parse leniency.** The `strip_prefix(&prefix)` + `parse::<usize>()`
   path at chat_composer.rs:1647-1651 silently ignores any `[Pasted
   Content N chars] #garbage` placeholder. That's fine for the current
   producer (only this function emits placeholders), but if a third
   party ever inserts a label like `[Pasted Content 100 chars] #v2`,
   the new code returns `[Pasted Content 100 chars]` (suffix=0) and
   collides with an existing bare placeholder. Probably acceptable —
   nothing else writes placeholders today — but a debug-assert that
   the suffix parses cleanly would catch future producers stepping on
   the format.

3. **No `Drop`/`Clear` test for `pending_pastes` itself.** The test
   relies on `composer.handle_key_event(Ctrl-C)` (or the equivalent)
   actually clearing `pending_pastes`. If a future refactor leaves
   stale entries in `pending_pastes` after Ctrl+C, this test would
   keep passing on a different code path. A direct
   `assert!(composer.pending_pastes.is_empty())` after the clear
   would lock that contract down.

4. **No-op rename: `next_suffix` → `max_suffix`.** Renaming the local
   reflects the new semantics ("max in pending pastes" rather than
   "next monotonic counter"), good. Worth a one-line comment that
   `max_suffix == 0` means "no same-size placeholder is currently
   pending," to save the next reader a moment.

## Verdict

**merge-as-is.** Replaces a stale counter with a derived value from
the live state — exactly the right shape for "label-allocation that
needs to reuse labels after deletion." Test pins the original
failure mode. The struct gets smaller and a `&mut self` becomes
`&self`. Hard to argue with.
