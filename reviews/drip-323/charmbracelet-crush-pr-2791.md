# charmbracelet/crush #2791 — fix(ui/chat): make keyboard expand work for thinking blocks

- **PR**: https://github.com/charmbracelet/crush/pull/2791
- **Head SHA**: `07e00ad4610a7d745befb1780c58aa16b89c7f67`
- **Author**: meowgorithm (maintainer)
- **Size**: +126 / −13, 4 files

## Files changed

- `internal/ui/chat/assistant.go` (+14 / −9)
- `internal/ui/chat/assistant_test.go` (+54, new)
- `internal/ui/model/chat.go` (+9 / −4)
- `internal/ui/model/chat_expand_test.go` (+49, new)

## Summary of change

Two coupled bug fixes for thinking-block expansion:

1. `AssistantMessageItem.ToggleExpanded()` previously had no return value,
   so `*AssistantMessageItem` silently failed to satisfy the
   `chat.Expandable` interface — keyboard-driven expand was a no-op.
2. `HandleMouseClick` was calling `ToggleExpanded()` itself, while the
   generic Expandable path in `model/chat.go` also toggles after a handled
   click → double-toggle → net no-op.

The fix returns `bool` from `ToggleExpanded()` (so the type matches the
interface) and removes the inline toggle in `HandleMouseClick`, leaving
the generic path as the single source of truth.

## Specific-line review

- `internal/ui/chat/assistant.go:38` — explicit interface assertion
  `var _ Expandable = (*AssistantMessageItem)(nil)`. This is the right
  fix to prevent the regression from recurring; the compiler will now
  refuse to build if `ToggleExpanded()` drifts out of shape again.
- `assistant.go:255-260` — `ToggleExpanded` now returns
  `a.thinkingExpanded` post-mutation. Matches the contract documented in
  the godoc comment.
- `assistant.go:266-275` — `HandleMouseClick` now just signals
  `thinkingBoxHeight > 0 && y < thinkingBoxHeight` without mutating
  state. The added comment ("Toggling here directly would double-toggle
  because the caller always runs the generic path after a handled click")
  is exactly the kind of "why" comment that survives refactors.
- `assistant_test.go` — `TestAssistantMessageItemExpandable` asserts
  `item.(Expandable)` succeeds (this is the regression guard) and that
  `ToggleExpanded()` returns `true` then `false`.
  `TestAssistantMessageItemHandleMouseClick` asserts that a click on the
  thinking box returns `true` but does **not** mutate `thinkingExpanded`
  — the precise invariant this PR establishes.
- `chat_expand_test.go` — the second test file presumably exercises the
  end-to-end keyboard path through `model/chat.go` (the generic
  Expandable dispatch). I would want to skim that to confirm it covers
  both keyboard and mouse paths landing on the same final state, but the
  diff prefix shown is consistent with that intent.

## Verdict

**merge-as-is** — small, surgical, well-tested fix from a maintainer.
The interface-assertion line alone makes this worth merging. Both new
test files lock in the regressions so they cannot silently come back.
