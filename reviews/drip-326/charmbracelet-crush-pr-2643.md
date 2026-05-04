# charmbracelet/crush PR #2643 — fix: enable real-time reasoning display and implement missing toggle handler

- **PR:** https://github.com/charmbracelet/crush/pull/2643
- **Author:** pi128
- **Head SHA:** `4959761a` (full: `4959761a5ae70c32f162b3d476b432152e48e353`)
- **State:** OPEN
- **Files touched:**
  - `internal/ui/chat/assistant.go` (+19 / -8) — bypass cache while spinning, gate thinking on `showThinking`
  - `internal/ui/model/ui.go` (+35 / -6) — `showThinking` field, `refreshMessages` helper, plumb through `ExtractMessageItems`
  - `internal/ui/chat/messages.go` (+3 / -2) — `ExtractMessageItems` signature change
  - `internal/ui/dialog/commands.go` (+5 / -0) — register "Hide/Show Thinking" command
  - `internal/ui/dialog/actions.go` (+1 / -0)
  - `internal/cmd/session.go` (+1 / -1) — pass `true` for non-interactive output

## Verdict

**merge-after-nits**

## Specific refs

- `internal/ui/chat/assistant.go:88-103` — the cache-bypass-while-spinning fix. The bug analysis in the PR body is correct: `RawRender` previously cached on width alone, so the first call during streaming caches an empty render and every subsequent frame hits stale cache. Bypassing on `isSpinning()` is the minimal correct fix; a smarter alternative would be invalidating cache on `SetMessage` but bypass-while-streaming has the lower complexity ceiling.
- `internal/ui/chat/assistant.go:142-148` — gating thinking render on `a.showThinking` is straightforward.
- `internal/ui/chat/messages.go:266-280` — `ExtractMessageItems` gains a positional `showThinking bool` param. **API break** for any external caller (and there are 3 internal call sites in `ui.go` plus `session.go`'s non-interactive output path). Consider an options struct (`ExtractOptions{ShowThinking: true}`) so future flags don't keep extending the positional list.
- `internal/ui/model/ui.go:333` — `showThinking: true` default preserves current behavior. Good.
- `internal/ui/model/ui.go:898-910` — `refreshMessages` re-reads `ListMessages` from the DB to apply the toggle. That round-trip is wasteful when the only thing that changed is a render flag. A pure re-render of the cached message list would be lighter.
- `internal/ui/dialog/commands.go:457-461` — `toggle_thinking_visibility` registered conditionally on `c.hasSession`. Note there's already an `ActionToggleThinking` (model-side feature toggle) and now also `ActionToggleThinkingVisibility` (UI display) — easy for users to confuse. Worth adding a clarifying description on the new command (e.g., "Hide reasoning content from chat view (does not affect generation)").

## Rationale

Two real bugs fixed: streaming reasoning never appearing, and a previously-declared toggle that did nothing. The cache-bypass approach is the right tactical fix. The signature change is the main thing to flag — positional bools are a known anti-pattern and this code path will keep accumulating render flags. Either land an options struct now or accept that a future PR will refactor.
