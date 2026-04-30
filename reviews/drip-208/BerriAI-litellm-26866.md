# BerriAI/litellm#26866 — feat(pricing): add gpt-image-2 and azure_ai/gpt-image-2 model pricing

- PR: https://github.com/BerriAI/litellm/pull/26866
- Head SHA: `02d1ef3c8c...`
- Author: xodn348
- Files: 2 changed (`model_prices_and_context_window.json` + `litellm/model_prices_and_context_window_backup.json`), +28 / 0

## Context

OpenAI shipped gpt-image-2 (and Azure mirrors it as `azure_ai/gpt-image-2`); litellm's pricing table didn't have entries for either, so cost calculation falls through to the `0.0` default and routing for `/v1/images/generations` and `/v1/images/edits` doesn't pick up the model. This PR adds the two table entries.

## Design

Identical structure for both entries, only `litellm_provider` and `source` differ:

```json
"gpt-image-2": {
    "cache_read_input_image_token_cost": 2e-06,
    "cache_read_input_token_cost": 1.25e-06,
    "input_cost_per_image_token": 8e-06,
    "input_cost_per_token": 5e-06,
    "litellm_provider": "openai",
    "mode": "image_generation",
    "output_cost_per_image_token": 3e-05,
    "source": "https://platform.openai.com/docs/models/gpt-image-2",
    "supported_endpoints": [
        "/v1/images/generations",
        "/v1/images/edits"
    ]
},
```

The Azure entry at `model_prices_and_context_window.json:2078-2091` is the same numbers with `litellm_provider: "azure_ai"` and a `source` pointing at the Azure pricing page. Both files (the canonical and the `_backup` copy) are kept in sync, which matches the convention.

## What's good

- **Token-based pricing fields are correctly broken out into the four kinds** the rest of litellm's image-pricing pipeline expects: `input_cost_per_token` (text prompt), `input_cost_per_image_token` (input image tokens), `output_cost_per_image_token` (generated image tokens), and `cache_read_input_image_token_cost` / `cache_read_input_token_cost` for the cached paths. This is the same field set as the existing `gpt-image-1` entry, so cost calculators will pick it up without code changes.
- **`supported_endpoints` includes both `/v1/images/generations` and `/v1/images/edits`** — important because gpt-image-2 actually supports edits (unlike DALL·E 2/3), and missing this would silently route edits requests to a fallback that doesn't price correctly.
- Both files updated together — someone running with the bundled `_backup` copy as a fallback (e.g., air-gapped deploys) gets the same pricing.

## Risks / nits

- **Cross-check against the upstream docs page** — the per-token numbers (`5e-06` input text, `8e-06` input image, `3e-05` output image, `1.25e-06` cache read text, `2e-06` cache read image) need to match what's at https://platform.openai.com/docs/models/gpt-image-2 *at the time this is reviewed*. I can't verify the price page from here. Reviewer with access should spot-check; if OpenAI quotes per-1M-token, divide by 1e6 and confirm. The PR body should ideally include the source quote for audit.
- **No `cache_creation_input_token_cost` entry** — gpt-realtime and other recent OpenAI models include this for the cache-write path. If gpt-image-2 supports prompt caching (which the presence of `cache_read_input_*` implies it might), a `cache_creation_*` field is likely also needed. If gpt-image-2 has no separate cache-creation cost (i.e., cache writes are free or rolled into input), explicitly note that in the source. This is the same correctness gap the recent #26872/#26883 PRs fixed for the model-DB and custom-pricing paths — easy to leak silently if the table only has the read-side fields.
- **No companion test** — the rest of the litellm pricing-PR convention is to add a regression test in `tests/test_litellm/cost_calculator/` that pins the exact dollar amount for a representative request. Worth adding so that if someone touches the parsing pipeline these exact numbers are protected.
- **`gpt-image-2` ordering** in the JSON is alphabetical between `gpt-image-1` and `gpt-realtime`, good. The `azure_ai/gpt-image-2` entry slots between `azure/...` and `azure_ai/gpt-oss-120b`, also good.

## Verdict

**merge-after-nits** — the entries are structurally correct and match the existing image-pricing convention. Before merge:

1. **Verify the five per-token numbers** against the live OpenAI/Azure pricing pages (PR body should quote the source numbers for audit).
2. **Confirm `cache_creation_input_token_cost` is genuinely zero/absent** for gpt-image-2 prompt caching, not just forgotten. This is the same gap that bit #26872 and #26883.
3. Add a one-line cost-calculator test pinning a representative request → expected dollar amount.
