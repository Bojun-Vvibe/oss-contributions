# Review: openai/codex #20746

- **PR:** openai/codex#20746 — `Validate /goal objective length in TUI`
- **Head SHA:** `ec74cd2ae7e451acecedc09f18ab252dbe10a952`
- **Files:** 7 (+276/-0)
- **Verdict:** `merge-as-is`

## Observations

1. **Clean module split** — New `chatwidget/goal_validation.rs` (+64) holds the goal-specific validation; `slash_dispatch.rs` only adds the call sites (live path at line 484-489 and queued path at line 673-680). This keeps `slash_dispatch.rs` focused on dispatch orchestration as the PR description claims.

2. **Pasted-content expansion is handled correctly** — `goal_objective_with_pending_pastes_is_allowed` calls `ChatComposer::expand_pending_pastes` only when `pending_pastes.is_empty()` is false, avoiding the allocation/clone in the common path. Char counting via `.chars().count()` (not `.len()`) is correct for unicode display semantics.

3. **Source-aware composer reset** — `goal_objective_char_count_is_allowed` only clears the composer + drains pending submission state when `source == Live`. The `Queued` path correctly leaves composer state alone (the queue is draining and the composer has already moved on). Good detail.

4. **Snapshot test + 187-line `tests/goal_validation.rs`** cover the three documented scenarios (live, queued, pasted) plus the 4000-char boundary. The error string is locked in via the `.snap` file which will catch any accidental message rewording. The combined `MAX_THREAD_GOAL_OBJECTIVE_CHARS` import (4000) and `MAX_USER_INPUT_TEXT_CHARS` import suggests the test also exercises the interplay between the two limits — appropriate.

## Recommendation

This is exactly how a TUI-side preflight should look. Ship it.
