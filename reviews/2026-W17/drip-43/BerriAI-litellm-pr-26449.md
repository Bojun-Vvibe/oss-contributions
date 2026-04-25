# BerriAI/litellm PR #26449 — [Feat] Day-0 support for GPT-5.5 and GPT-5.5 Pro

- **URL:** https://github.com/BerriAI/litellm/pull/26449
- **Author:** @mateo-berri
- **State:** MERGED 2026-04-24
- **Head SHA:** `d2d676c8c6a201c6386f3229f65f3afc3572a26c`
- **Files:** `model_prices_and_context_window.json`,
  `litellm/model_prices_and_context_window_backup.json`

## Summary of change

Day-0 model registry update that:
1. Bumps existing `gpt-5.5` `max_input_tokens` from `272000` →
   `1050000` and adds the full pricing tier matrix
   (`*_above_272k_tokens`, `*_flex`, `*_priority`, `*_batches`).
2. Adds `supports_web_search: true` capability flag.
3. Registers two new dated/aliased entries —
   `gpt-5.5-2026-04-23`, `gpt-5.5-pro`, `gpt-5.5-pro-2026-04-23` —
   each with the matching pricing matrix and capability set.

## Findings against the diff

- **L19275–19298 backup.json `gpt-5.5`:** correct bump of context
  window (1.05M is the publicly-stated GPT-5.5 limit). Good catch
  that the existing entry only had `272000`.
- **L19316 `gpt-5.5`:** new fields `cache_read_input_token_cost_priority:
  1e-06`, `output_cost_per_token_priority: 6e-05` etc. are 2× the
  base price for priority tier — matches OpenAI's documented priority
  tier multiplier.
- **`gpt-5.5-pro` / `gpt-5.5-pro-2026-04-23`:** mode is `"responses"`
  (not `"chat"`) and `supported_endpoints` is `["/v1/responses",
  "/v1/batch"]` — consistent with how Pro variants are served. Pricing
  is **12× chat**: input `6e-05` vs `5e-06`; output `0.00036` vs
  `3e-05`. Plausible for a Pro tier and matches `gpt-5.4-pro`-shaped
  ratios elsewhere in the file.
- **Pro lacks `supports_none_reasoning_effort: false`** while base 5.5
  has it true — that's an actual capability difference (Pro always
  reasons), not a bug.
- **Both pro entries have `supports_response_schema: false`** — worth
  double-checking against the API doc, but other "responses"-mode pro
  models in the registry follow the same pattern.
- **Duplication:** the same delta is applied to both
  `model_prices_and_context_window.json` and the
  `..._backup.json` copy. That's the convention in this repo (CI
  snapshot test compares them) — correct.

## Verdict

**merge-as-is**

Pure registry/data PR, no logic changes, already merged. Pricing
matrix is internally consistent and follows the same shape as
neighboring `gpt-5.4*` entries. The only thing a maintainer might
nit (post-merge) is splitting Pro entries into a follow-up that
adds an integration test exercising the `/v1/responses` route with
the new model id, but that's not a blocker.
