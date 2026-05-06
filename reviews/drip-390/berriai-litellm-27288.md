# Review: BerriAI/litellm#27288 — feat(pricing): add Voyage v4 embedding model pricing

- Head SHA: `a554b599bee92c4484a9fb171fa54e60d54cdc79`
- Files: 1 (+24 / -0)
- Verdict: **merge-after-nits**

## Summary

Adds three Voyage AI v4 embedding entries (`voyage/voyage-4-large`, `voyage/voyage-4`, `voyage/voyage-4-lite`) to `model_prices_and_context_window.json`. Pure data-file addition, no code path touched. Closes the gap where v4 spend is invisible to the cost tracker because the keys don't exist in the bundled price table.

## Specific evidence

- **Three identically-shaped entries** at `model_prices_and_context_window.json:33831-33854`:
  ```json
  "voyage/voyage-4-large": {
      "input_cost_per_token": 1.2e-07,
      "litellm_provider": "voyage",
      "max_input_tokens": 32000,
      "max_tokens": 32000,
      "mode": "embedding",
      "output_cost_per_token": 0.0
  },
  "voyage/voyage-4":       { ... 6e-08 ... },
  "voyage/voyage-4-lite":  { ... 2e-08 ... }
  ```
  Schema matches the existing `voyage/voyage-code-2` entry immediately following, alphabetically inserted between `voyage/voyage-3-multilingual` (or similar) and `voyage/voyage-code-2`. `output_cost_per_token: 0.0` is correct for embedding models (no generated tokens). `mode: "embedding"` is the gate that routes through `aembedding`/`embedding` cost lookups.
- **Pricing tier shape is monotonic and matches the published Voyage tier ladder**:
  - large = $0.12/1M (1.2e-07/token)
  - mid   = $0.06/1M (6e-08/token)  ← exactly half of large
  - lite  = $0.02/1M (2e-08/token)  ← one-third of mid, one-sixth of large
  PR body cites `https://docs.voyageai.com/docs/pricing` for sourcing.
- **32k context window** for all three matches the v4 family spec.

## Nits

1. **Test plan checkboxes are unchecked** (`- [ ]` at both bullets). The two checks are trivial JSON-validity / lookup-by-key — a maintainer should run them or have a CI check that diffs the file for parse-validity. The latter likely already exists since this file is amended on every model addition, but worth confirming.
2. **No comparison to existing `voyage-3-large` entry** in the PR body — for Voyage's tier-stable history (v3-large is $0.18/1M per their old pricing), the v4-large drop to $0.12/1M is a ~33% reduction. Worth one line in the PR body for downstream cost-model audits.
3. **`output_cost_per_token: 0.0` not `null` or omitted** — consistent with sibling entries, but a future schema migration could prefer absence over zero. Non-blocking, ecosystem-wide convention.
4. **No deprecation marker on v3.5 entries** (the PR body states v4 supersedes v3.5) — separate concern, can be a follow-up. Removing `voyage-3.5*` prematurely would break callers; deprecation_date metadata would be the correct path.
5. PR body references "Generated with Claude Code" — that watermark is acceptable on a pricing-data PR but the maintainer's PR-template likely doesn't require attribution.

Single-file pricing add with correct schema and externally-cited tier ratios. Merge after the two test-plan boxes are checked.
