# cline/cline #10401 — feat(deepseek): Add deepseek-v4-flash and deepseek-v4-pro support

- **Repo**: cline/cline
- **PR**: [#10401](https://github.com/cline/cline/pull/10401)
- **Head SHA**: (head commit on the `feat/deepseek-v4` branch)
- **Author**: gerryqi
- **State**: OPEN (+23 / -1)
- **Verdict**: `request-changes`

## Context

Adds two new DeepSeek model entries to the static catalog at
`src/shared/api.ts:2122-2147`. Pure metadata addition — no
provider code change, no UI change. Both models exist on the
DeepSeek platform; this surfaces them in Cline's model picker
with their context window, pricing, and capability flags.

## Design

Adds two map entries inside `deepSeekModels`:

1. `"deepseek-v4-flash"` (lines 2124-2132): 8K maxTokens / 64K
   context, no images, prompt cache enabled, `inputPrice: 0`,
   output `0.28`, cache writes `0.55` ... wait, reading more
   carefully: writes `0.07`, reads `0.028`. The `inputPrice: 0`
   is suspicious — see Risks.

2. `"deepseek-v4-pro"` (lines 2133-2143): 384K maxTokens / 1M
   context, no images, prompt cache enabled, `inputPrice: 0
   // uses cache-based pricing`, output `3.48`, cache writes
   `1.74`, cache reads `0.145`. Also flips
   `supportsReasoning: true` and `supportsReasoningEffort: true`
   on this entry only — V4-pro is the reasoning model, flash is
   not.

3. Type-annotation widening on the closing line:
   `} as const satisfies Record<string, ModelInfo>` becomes
   `Record<string, OpenAiCompatibleModelInfo>`. This is the only
   real correctness lever in the PR — the `supportsReasoningEffort`
   field exists on `OpenAiCompatibleModelInfo` but not on
   `ModelInfo`, so without the widening the new pro entry would
   fail typecheck. The cost: every other entry in
   `deepSeekModels` is also re-typed to the wider interface,
   which weakens the contract on existing models.

## Risks

1. **`inputPrice: 0` is almost certainly wrong** for both models.
   DeepSeek's published pricing for V4-flash is non-zero per
   million input tokens; the comment `// uses cache-based pricing`
   on V4-pro hints the author meant "we'll account for it via
   cache pricing" but Cline's cost calculator multiplies
   `inputPrice * inputTokens` for the *uncached* portion of every
   request. A user whose first request to a V4 model has 50K
   uncached input tokens will see `$0.00` cost, which is a
   reporting bug at minimum and a billing-shock landmine for
   users who make decisions based on Cline's cost panel.
2. **No source citation.** The PR adds prices and context windows
   but doesn't link to the DeepSeek pricing page or release
   notes. For a metadata change that drives billing math,
   reviewers can't verify the numbers without out-of-band
   research. Standard practice in this file (see
   `deepseek-chat` entry above) is to either link or cite a
   commit-time snapshot.
3. **Type widening side-effect.** Switching the catalog type to
   `OpenAiCompatibleModelInfo` removes type-level guarantees
   that `deepseek-chat` and friends *don't* accidentally start
   carrying OpenAI-compat-only fields. Better path: keep the
   map as `Record<string, ModelInfo>` and widen *only* the V4
   entries via a per-entry `satisfies OpenAiCompatibleModelInfo`,
   or extract a `DeepSeekModelInfo = ModelInfo &
   Partial<Pick<OpenAiCompatibleModelInfo, "supportsReasoningEffort">>`
   so the surface widens precisely.
4. **`maxTokens: 384_000` on a 1M context model** for V4-pro is
   plausible (output cap < context window) but worth confirming
   — DeepSeek publishes the output-token ceiling separately and
   Cline's prompt builder uses `maxTokens` as the response budget.
5. **`supportsImages: false` on both.** Confirm against
   DeepSeek's announcement — if either model supports image
   input, this entry will silently strip images from prompts at
   the provider layer.

## Suggestions

1. **Cite the pricing source** in the PR body or as a comment.
   Link DeepSeek's pricing page and the date the snapshot was
   taken.
2. **Fix `inputPrice: 0`**. If the model truly bills only via
   cache reads/writes (no uncached input charge), document that
   explicitly in a comment AND verify Cline's cost calculator
   handles the `cache miss → uncached input` accounting
   correctly. Otherwise put the published per-token price in.
3. **Narrow the type widening**. Don't relax the whole map's
   type just to land one field on one entry.
4. **Add a screenshot or test** showing the new entries appear
   in the picker and the cost panel produces a non-zero number
   on a non-trivial request.

## What I learned

Static model catalogs are deceptively dangerous PRs: the diff
looks like a config blob, but every number is wired into a cost
calculator, a prompt budget, or a capability gate somewhere
downstream. The "inputPrice: 0" pattern in particular needs
guardrails — either the type system rejects 0 for non-free
models, or the cost calculator surfaces a "pricing unknown"
warning instead of $0. Worth a follow-up issue regardless of
this PR's outcome.
