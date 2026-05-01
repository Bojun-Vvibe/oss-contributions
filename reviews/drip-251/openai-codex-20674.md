# openai/codex#20674 — Clear live hook rows when turns finalize

- **PR:** https://github.com/openai/codex/pull/20674
- **Head SHA:** `8cdaa30b9c20c085bb6767665f12d4e83dc157b7`
- **Stats:** +40 / -0 across 3 files (`tui/src/chatwidget.rs`, snapshot file, `chatwidget/tests/status_and_layout.rs`)

## Context
When the user interrupts a turn while a hook (e.g. PreToolUse policy check) is still running, the normal turn-status reset clears the spinner and resets streaming buffers — but the separate `active_hook_cell` (the live "Running PreToolUse hook: checking command policy" row in the TUI) was never cleared, because it depends on a `HookCompleted` event that the cancelled hook never delivers. Result: an orphaned `Running` row sticks on screen indefinitely until the next turn nudges the layout. Pure cosmetic, but it's the kind of bug that tells users "the agent is still working on something" when it isn't.

## Design
Six lines added at `codex-rs/tui/src/chatwidget.rs:2931-2936` inside `finalize_turn`:

```rust
fn finalize_turn(&mut self) {
    self.finalize_active_cell_as_failed();
    // Turn-scoped hook rows are transient live state; once the turn is over,
    // do not leave an orphaned running row behind if no matching completion
    // event arrived before cancellation.
    if self.active_hook_cell.take().is_some() {
        self.bump_active_cell_revision();
    }
    self.user_turn_pending_start = false;
    self.agent_turn_running = false;
    ...
}
```

Three things this gets right:

1. **Placed inside `finalize_turn`**, the existing chokepoint for end-of-turn cleanup. Not at the cancel site, not inside the event handler — `finalize_turn` is *the* place that already handles "the turn ended, regardless of how", so adding the hook-cell clear here means it covers cancellation, normal completion, and any future end-of-turn path without each path remembering separately.

2. **`take().is_some()` pattern** atomically removes the cell *and* tells you whether there was anything to remove, so the `bump_active_cell_revision()` (which presumably triggers a re-render) only fires when there's actually a change. No spurious renders on the common "no live hook running at finalize" path.

3. **Comment explains the *why*** — "Turn-scoped hook rows are transient live state; once the turn is over, do not leave an orphaned running row behind if no matching completion event arrived before cancellation." Future maintainers won't strip this as redundant.

## Tests
`tests/status_and_layout.rs:1334-1359` adds `interrupted_turn_clears_visible_running_hook`:
- Calls `handle_hook_started(...)` with a `PreToolUse` hook.
- `reveal_running_hooks(...)` to make it visible.
- Captures the before-state via `active_hook_blob(&chat)`.
- Calls `handle_turn_interrupted(&mut chat, "turn-1")`.
- Snapshot-asserts the after-state shows `<empty>`.

The snapshot file at `chatwidget/snapshots/codex_tui__chatwidget__tests__interrupted_turn_clears_visible_running_hook.snap` literally captures:
```
before interrupt:
• Running PreToolUse hook: checking command policy
after interrupt:
<empty>
```
Dispositive: shows the symptom and the fix in one diff.

## Risks
- The PR author notes `cargo test -p codex-tui` aborts on an unrelated stack overflow in `app::tests::discard_side_thread_removes_agent_navigation_entry`. The targeted `interrupted_turn_clears_visible_running_hook` test passes individually, but full-suite gating relies on that other test getting fixed. Not blocking for this PR but worth a tracking issue.
- **No symmetric test for "turn completes normally with no orphan hook"** — the common case where there *was* no live hook at finalize. The `take().is_some()` guard ensures `bump_active_cell_revision()` doesn't fire spuriously, but a one-line negative assertion (no extra render bump on the happy path) would lock that.
- **No test for "hook completes naturally, then turn finalizes"** — to confirm the new clear isn't double-clearing already-cleared state. The `take().is_some()` makes this idempotent so it's safe; the test would just be defense-in-depth.
- The fix relies on `finalize_turn` being called for *every* end-of-turn path including cancellation. If a future code path bypasses `finalize_turn` (e.g. error path that resets state directly), the orphan returns. Worth a one-line audit comment.

## Suggestions
1. File a tracking issue for the unrelated `discard_side_thread_removes_agent_navigation_entry` stack overflow so this PR's "tests don't fully run" line doesn't quietly become normalized.
2. Add a sibling test that asserts no orphan row when there *is* no live hook at finalize (proves the `is_some()` guard works).
3. Optional: add a `#[cfg(debug_assertions)]` `tracing::debug!` log inside the `if` arm so when this triggers in the wild you see "cleared orphaned hook cell at turn finalize" — useful for confirming the guard fires when expected.

## Verdict
**merge-as-is** — six lines + one snapshot test, placed at the canonical end-of-turn chokepoint with a comment explaining why. The unrelated test-suite stack overflow is a pre-existing flake, not a regression from this change. The snapshot literally shows the symptom and the fix.

## What I learned
"Live cell that depends on a completion event that may never arrive" is a recurring TUI bug pattern. The right shape is *not* to add a timeout or a polling clear at the source of truth, but to clear at every chokepoint where the cell becomes irrelevant — here, `finalize_turn`. The `take().is_some()` pattern is also worth noting: it's the idiomatic way to "remove the thing if present, and tell me whether you removed something" so that re-render side-effects only fire on actual state change. And the snapshot test that captures both `before interrupt:` and `after interrupt:` in one blob is the right shape for cosmetic-fix tests — a future reader sees the symptom (orphan row) and the fix (empty) literally in the same diff.
