---
pr: 2643
repo: charmbracelet/crush
sha: 4959761a5ae70c32f162b3d476b432152e48e353
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2643 — fix: enable real-time reasoning display and implement missing toggle handler

- **Author**: (community contributor)
- **Head SHA**: 4959761a5ae70c32f162b3d476b432152e48e353
- **Size**: +64/-17 across `internal/cmd/session.go`, `internal/ui/chat/{assistant,messages}.go`, `internal/ui/dialog/{actions,commands}.go`, `internal/ui/model/ui.go`.

## Scope

Two coupled bugs in TUI reasoning rendering:

1. **Real-time display broken**: `AssistantMessageItem.RawRender()` cached rendered content keyed only by width. First render captured empty/incomplete reasoning content, all subsequent renders during the streaming phase hit cache and returned the stale empty content. Fix: bypass cache while `isSpinning()` is true, render fresh every frame; only cache once the assistant turn is finalized.

2. **Toggle handler missing**: there was a "Hide/Show Thinking" command-palette entry intent (`ActionToggleThinkingVisibility`) but no `showThinking` field on `UI` and no propagation through `ExtractMessageItems → NewAssistantMessageItem`. PR adds `showThinking` (default `true`) on `UI`, threads it through every `ExtractMessageItems` call site, and respects it in `AssistantMessageItem.renderMessageContent`.

## Specific findings

- `internal/ui/chat/assistant.go:91-105` — the cache-bypass during streaming is the right fix. The previous `getCachedRender(cappedWidth)`-first path is preserved for the post-stream finalized state, so the cache benefit (avoiding re-render on width-stable redraws) survives. Reads cleanly.
- `internal/ui/chat/assistant.go:144-148` — `if thinking != "" && a.showThinking` gate on the `renderThinking` block is the right shape. The `thinking != ""` short-circuit means a model that simply doesn't produce reasoning content is unaffected by the toggle (no empty header rendered when off).
- `internal/ui/chat/messages.go:265-280` — the `ExtractMessageItems` signature gains a trailing `showThinking bool` parameter. **API break for any external caller** of `ExtractMessageItems` — but the function is in `internal/`, so no external callers exist. Acceptable.
- `internal/cmd/session.go:480` — passes `true` unconditionally to `ExtractMessageItems` in the human-output path (`crush session show ... --human` or similar). Correct: when dumping a session for human consumption, you want reasoning content visible by default. Worth adding a `--no-thinking` CLI flag in a follow-up so users can suppress reasoning in scripted output.
- `internal/ui/model/ui.go:253, 333, 901-913` — `showThinking` field added with default `true` (matches prior behavior — toggle off → matches previous always-show; toggle on → unchanged). The new `refreshMessages` helper at `:901-913` re-runs `ListMessages` from the database to rebuild the items list when toggled. **Subtle issue**: rebuilding from DB drops the in-memory `cachedRender` of every assistant item that was already finalized. Toggling thinking visibility will force a full re-layout of the chat history. For a long session this could be slow. Consider a lighter "walk existing items, set `showThinking` and invalidate cache" path instead of full DB re-fetch.
- `internal/ui/model/ui.go:933-1041` — every `chat.ExtractMessageItems(...)` call site (initial load, append, nested tool-calls) now passes `m.showThinking`. Consistent. But missed call sites would leave the toggle half-broken — verify no `ExtractMessageItems` call exists outside `internal/ui/model/ui.go` and `internal/cmd/session.go`.
- `internal/ui/dialog/{actions,commands}.go` — `ActionToggleThinkingVisibility` action defined and a "Hide/Show Thinking" command item conditionally added when `c.hasSession` is true. Good — the command is hidden when no session exists (where it would no-op). Missing in the diff: the *handler* wiring that maps `ActionToggleThinkingVisibility` to `m.showThinking = !m.showThinking; return m.refreshMessages()`. Either it's outside the head-200 window of the diff or it's the bug the PR title alludes to ("implement missing toggle handler") and ships in the part I haven't seen — verify before approving.
- **No test**. Toggle-state correctness, cache-bypass-while-spinning, and post-finalize-cache-restore are all unit-testable; none have coverage in the diff.

## Risk

Low-medium. The cache-bypass-while-spinning is a clear correctness fix and unlikely to regress existing behavior. The toggle plumbing touches many call sites, so the audit risk is "did the author miss a call site?" — easily answered with `git grep ExtractMessageItems`. The refresh-from-DB on toggle is a perf concern, not a correctness one.

## Verdict

**merge-after-nits** — (1) confirm the action handler that maps `ActionToggleThinkingVisibility` → state mutation actually exists in this PR (or extend the diff); (2) replace the DB re-fetch in `refreshMessages` with an in-memory item rebuild; (3) add a smoke test for the cache-bypass behavior (assert that two `RawRender` calls with the same width but different `message.ReasoningContent().Thinking` produce different output during `isSpinning`); (4) add a `--no-thinking` flag to the human session dump as a follow-up. The core fix is correct.
