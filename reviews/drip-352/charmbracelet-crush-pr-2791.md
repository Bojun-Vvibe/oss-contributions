# charmbracelet/crush #2791 — fix(ui/chat): make keyboard expand work for thinking blocks

- URL: https://github.com/charmbracelet/crush/pull/2791
- Head SHA: `07e00ad4610a7d745befb1780c58aa16b89c7f67`
- Author: meowgorithm
- Size: +126 / -13 across 4 files

## Comments

1. `internal/ui/chat/assistant.go:38` — `var _ Expandable = (*AssistantMessageItem)(nil)` is exactly the right guard — it's the compile-time witness that prevents the original bug (the type silently failing to satisfy `Expandable`). Keep this and consider adding the same line for every other `MessageItem` that should be expandable.
2. `internal/ui/chat/assistant.go:255-259` — `ToggleExpanded` now returns `bool`. Confirm no out-of-tree callers rely on the void signature; this is a minor API break for plugins that embed `chat`.
3. `internal/ui/chat/assistant.go:264-275` — The "double-toggle" comment is the highest-value comment in the diff: it documents *why* `HandleMouseClick` no longer calls `ToggleExpanded`. Excellent. Add a similar comment in `internal/ui/model/chat.go:606-614` referencing this file so future readers find both ends of the contract.
4. `internal/ui/model/chat.go:603-618` — Gating the `ToggleExpanded` call on `handled` is the actual fix. Verify `MouseClickable` implementers that *don't* implement `Expandable` (e.g. tool-call rows that toggle their own state inline) still behave: previously they got a free `ToggleExpanded` call that no-op'd via type assertion failure; now they explicitly skip it. Should be fine, but worth a manual click-through.
5. `internal/ui/chat/assistant_test.go:24-32` — `TestAssistantMessageItemExpandable` is exactly the regression test the original bug needed. The `require.True(t, ok, "...must satisfy Expandable")` assertion will fire at test time if anyone reverts the return value, which is the goal.
6. `internal/ui/model/chat_expand_test.go:19-49` — Integration-level test that wires a real `Chat` and exercises the keyboard path. The "first toggle expands, direct re-toggle returns false" pattern is clever but a touch indirect; a comment at the top of the file (already present, very thorough) makes it readable. Good.
7. Consider also wiring this through to a snapshot test of the rendered frame so a future styles refactor that changes `thinkingBoxHeight` doesn't silently re-break the click hit-test.

## Verdict

`merge-as-is`

## Reasoning

This is a textbook bug-fix PR: tightly scoped to the regression, rooted in a missing return value that broke an interface contract, fixed at both the implementer (return `bool`) and caller (gate on `handled`), with a compile-time witness (`var _ Expandable = ...`) and two layers of test (unit + integration) that explicitly call out the original bug in their docstrings. The code, the comments, and the tests all reinforce each other. Minor follow-ups (snapshot test, plugin-API note) are nice-to-have, not blocking.
