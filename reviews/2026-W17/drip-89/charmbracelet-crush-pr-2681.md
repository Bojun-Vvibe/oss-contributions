---
pr: 2681
repo: charmbracelet/crush
sha: e2caa0ba34b024acf5429a40b818c4550d1e3f87
verdict: merge-after-nits
date: 2026-04-27
---

# charmbracelet/crush #2681 — fix(agent): retry title generation with large model on empty output

- **Head SHA**: `e2caa0ba34b024acf5429a40b818c4550d1e3f87`
- **Size**: focused on `internal/agent/agent.go`

## Summary
Fixes a real GPT-5-mini-class regression where reasoning-heavy "non-reasoning" models burn the entire output budget on hidden chain-of-thought and emit zero visible text for short prompts (title generation). Three changes:
1. Bumps `var maxOutputTokens int64 = 40` → constant `smallTitleMaxTokens int64 = 256` (`agent.go:57-60`) — gives the small model room for hidden reasoning before visible output.
2. Adds `fallbackTitleMaxTokens int64 = 1024` (`agent.go:62-64`) for the large-model retry path when `CatwalkCfg.DefaultMaxTokens` is zero.
3. After the existing small→large *error*-triggered fallback, adds a *second* retry path (`agent.go:994-1009`) for the case where the small model returned cleanly but with empty/whitespace text (`!isUsableTitleResponse(resp)`).

## Specific findings
- `agent.go:57-64` — extracting the constants is good. The 256/1024 values are arbitrary; document the source (40 was a magic number too, but 256 is a 6.4× bump that warrants a one-line justification — "enough for one-line title + GPT-5-mini reasoning headroom" works).
- `agent.go:941` — `maxOutputTokens := smallTitleMaxTokens` for the small-model first attempt; then `if smallModel.CatwalkCfg.CanReason { maxOutputTokens = smallModel.CatwalkCfg.DefaultMaxTokens }`. Logic preserved. OK.
- `agent.go:976` — error message tag changed from `"err"` to `"error"`. Cosmetic but inconsistent with the rest of the file (`grep -n '"err"'` likely shows other call sites still using the short form). Pick one convention and use it everywhere.
- `agent.go:980` — error fallback path now uses `cmp.Or(largeModel.CatwalkCfg.DefaultMaxTokens, fallbackTitleMaxTokens)` instead of reusing `maxOutputTokens`. Correct fix; the prior code passed the small-model budget to the large-model agent.
- `agent.go:994-1009` — the new "retry on empty visible text" branch checks `model.CatwalkCfg.ID != largeModel.CatwalkCfg.ID && !isUsableTitleResponse(resp)`. The model-ID guard correctly avoids double-calling the same large model. Good.
- `agent.go:996` — the retry uses `cmp.Or(largeModel.CatwalkCfg.DefaultMaxTokens, fallbackTitleMaxTokens)`. Consistent with the error path.
- `agent.go:1003-1006` — only adopts the retry response if `isUsableTitleResponse(retryResp)` *also* returns true. Failing fall-through to the default-title path is the right safety net.
- **Missing diff context**: `isUsableTitleResponse()` is referenced but its definition isn't in the visible diff window. Verify it checks both nil-safety and `strings.TrimSpace(text) != ""` not just non-nil.
- **No test added** for the new branch. A unit-test fake `agent` returning empty text on first call and "Project X" on retry would lock in the contract. The test file is already extensive (`agent_test.go`); this is the right place.

## Risk
Low. The fallback chain only adds a second LLM call on a corner case (small model emits empty text without erroring). Worst case is a few extra LLM-hits per session creation for users on reasoning-heavy small models. Best case is the title actually populates.

## Verdict
**merge-after-nits** — add a unit test for the empty-response retry path; settle the `"err"` vs `"error"` log-key convention; one-line comment justifying the 256 constant.
