# cline/cline #10404 — feat(deepseek): deepseek-v4-pro supports reasoning effort control

- **PR:** https://github.com/cline/cline/pull/10404
- **Head SHA:** `0e78bf83fdbecf9d656b093cc2f32fd62fa48470`
- **Files changed:** 4 — `src/core/api/index.ts` (+1), `src/core/api/providers/deepseek.ts` (+19/−2), `src/shared/api.ts` (+22/−1), `webview-ui/src/components/settings/providers/DeepSeekProvider.tsx` (+13/−0).

## Summary

Adds `deepseek-v4-flash` and `deepseek-v4-pro` to the DeepSeek model registry, marks v4-pro as `supportsReasoning`/`supportsReasoningEffort`, threads `reasoningEffort` through the handler factory, and on v4-pro coerces the OpenAI-standard effort levels (`minimal`/`low`/`medium`/`high`/`xhigh`) down to the DeepSeek-specific binary (`high`/`max`). Adds a `ReasoningEffortSelector` to the settings UI when the selected DeepSeek model advertises `supportsReasoningEffort`.

## Line-level call-outs

- `src/core/api/providers/deepseek.ts:50-53` — `toDeepSeekReasoningEffort`:
  ```ts
  const normalized = normalizeOpenaiReasoningEffort(effort)
  return normalized === "xhigh" ? "max" : "high"
  ```
  This collapses `minimal`/`low`/`medium`/`high` → `high`, and only `xhigh` → `max`. So a user who sets the slider to `minimal` because they want a fast/cheap response gets the same `high` as a user who sets `medium`. That's silently wrong from the user's perspective: the UI tells them they're at `minimal` but the wire says `high`. Two cleaner choices: (a) clamp `minimal`/`low` → `high` *and* hide non-applicable levels in the UI for v4-pro, or (b) at minimum, log a one-time `console.warn` in the handler when an effort below `high` is normalized up so it shows up in support sessions. As written, "DeepSeek V4 Pro at minimal effort doesn't seem any cheaper" will be a real issue in the queue.
- `src/core/api/providers/deepseek.ts:32` — `type DeepSeekReasoningEffort = "high" | "max"` is declared but never re-exported, and the field on the wire is typed as `ChatCompletionReasoningEffort` via the cast on `:117`. The cast is needed because OpenAI's union doesn't include `"max"`. That cast is the right escape hatch, but worth a comment naming "DeepSeek V4 Pro accepts a non-OpenAI value here; cast required".
- `src/core/api/providers/deepseek.ts:68` — `const isDeepseekV4Pro = model.id.includes("deepseek-v4-pro")`. Substring match. If DeepSeek ever ships `deepseek-v4-pro-mini` (or any future `*-pro-*` variant) and it doesn't actually support reasoning effort, this silently injects `reasoning_effort: high` into every request. Tighter: equality check against the model registry, or a whitelist set. The same pattern at `:64` for `deepseek-reasoner` already has this risk; this PR repeats it rather than fixing it. Not blocking, but it's the kind of thing that breaks 6 months later when nobody remembers.
- `src/shared/api.ts:2138` — `inputPrice: 0, // uses cache-based pricing` for `deepseek-v4-pro`. The comment is good. Without context, downstream cost-summarization code that does `tokens * inputPrice` for non-cached input will report $0 input cost and confuse users. Worth confirming `calculateApiCostOpenAI` (imported in `deepseek.ts:2`) handles the "cache-only pricing" case — i.e., that all input usage is reported as `cacheReadsUsed` or `cacheWritesUsed` and never as plain input. If not, the cost UI will systematically under-report v4-pro spend.
- `src/shared/api.ts:2140` — `cacheReadsPrice: 0.145` for v4-pro vs `cacheWritesPrice: 1.74`. 12× ratio. Plausible for DeepSeek but worth one citation in the PR description (DeepSeek pricing page link with date) so a future maintainer can re-verify when prices move.
- `webview-ui/src/components/settings/providers/DeepSeekProvider.tsx:30-33` — `const showReasoningEffort = (modelInfo as any)?.supportsReasoningEffort === true`. The `as any` defeats the type system precisely at the type-narrowing point. Either widen `deepSeekModels`'s satisfied type to `OpenAiCompatibleModelInfo` (which the PR already does on `:113` with `satisfies Record<string, OpenAiCompatibleModelInfo>`), so the field is statically typed, or import `OpenAiCompatibleModelInfo` and cast properly. The `as any` is the only unsafe-cast in this PR and it's right next to the cleanup that would obviate it.
- `:55-61` — `<ReasoningEffortSelector defaultEffort="high" description="…" />` doesn't expose `availableEfforts`, so the selector likely shows the full OpenAI 5-level list. Combined with the silent collapse bug above, the user sees five options that map to two outputs. Either constrain the selector to `["high","xhigh"]` or rename the labels in the description.
- `src/core/api/index.ts:202` — `reasoningEffort: mode === "plan" ? options.planModeReasoningEffort : options.actModeReasoningEffort`. Correctly mirrors the `apiModelId` pattern on `:201`. Good.

## Verdict

**request-changes**

## Rationale

The wiring is structurally fine — the option threads from settings UI → handler factory → provider call site, and the cost/UI metadata lands in the right places. But the central piece, `toDeepSeekReasoningEffort`, silently maps four of the five OpenAI levels to the same DeepSeek output. That's a UX bug, not just a stylistic one: the user gets a control that does nothing for 80% of its range. Either constrain the UI to the two efforts DeepSeek actually accepts (cleanest), or be explicit that `minimal`/`low`/`medium` are clamped (with a visible warning the first time it happens). The substring-match model gate and the `as any` in the React layer are smaller; both worth one line each. Once the effort mapping reflects reality, this is merge-as-is.
