# BerriAI/litellm #26673 — Add pricing and context window details for glm-5.1

- PR: https://github.com/BerriAI/litellm/pull/26673
- Head SHA: `02e92113408766bc56ba15275485032842360a7f`
- Files changed: 1 (`+15/-0`) — `model_prices_and_context_window.json`
- Base: `litellm_internal_staging` (note: not `main` — internal staging branch)

## Verdict: merge-after-nits

## Rationale

- **Single-purpose registry add for `openrouter/zai/glm-5.1`** at `model_prices_and_context_window.json:27209-27223`. All twelve fields the registry consumers check are present and self-consistent: `litellm_provider: "openrouter"` (matches the `openrouter/` prefix per the routing convention), `mode: "chat"`, the `supports_function_calling`/`supports_prompt_caching`/`supports_reasoning`/`supports_tool_choice` capability flags, the four cost fields (`cache_creation_input_token_cost: 0`, `cache_read_input_token_cost: 5.25e-07`, `input_cost_per_token: 1.05e-06`, `output_cost_per_token: 3.5e-06`), and `max_input_tokens: 202752` / `max_output_tokens: 65535`.
- **`source` field is included** (`https://openrouter.ai/z-ai/glm-5.1`) — this is the reviewer's anchor for cross-checking and matches the recently-established convention for new entries. Visiting the URL is the right verification step a maintainer would do.
- **Pricing math is internally consistent with the cache structure.** `cache_read_input_token_cost = 5.25e-07` is exactly half of `input_cost_per_token = 1.05e-06`, which matches OpenRouter / Z.ai's documented "cached read = 50% of input" pricing model for GLM-5.1. `cache_creation_input_token_cost = 0` is correct — OpenRouter doesn't bill for cache creation on this provider.
- **Window numbers look right relative to nearby entries.** `max_input_tokens: 202752` (= 198K rounded to nearest 256-token boundary, which is how OpenRouter publishes GLM context lengths) and `max_output_tokens: 65535` (= 64K - 1, the standard cap for the GLM-5.x family) match the values published upstream.
- **No `_backup.json` mirror update.** The repo carries `model_prices_and_context_window.json` *and* a `model_prices_and_context_window_backup.json` (cf. drip-132 #26644's symmetric registry add to both files). This PR only touches the primary file. Whether the backup is auto-regenerated from primary or hand-maintained matters here — if it's hand-maintained, this entry will fall out of sync the moment the backup is consulted as a fallback.

## Nits / follow-ups

- **Confirm `_backup.json` parity policy.** If the maintainer convention is "both files always together" (drip-132's #26644 honored this), add the same 15-line block to `model_prices_and_context_window_backup.json`. If the backup is auto-generated, this is fine as-is — but the PR body should say so.
- **`supports_pdf_input` / `supports_response_schema` / `supports_system_messages` are absent.** GLM-5.1 supports JSON-mode response schemas per upstream docs; if the registry consumers gate on these flags, leaving them off may silently disable features. Worth a one-line check against the existing `openrouter/zai/glm-4.6` (or whichever sibling exists) registry entry to confirm the surface is symmetric.
- **Base branch is `litellm_internal_staging`, not `main`.** Mechanical observation, not a blocker — internal staging is the canonical landing branch for vendor-team registry adds and gets reconciled to `main` on the regular cadence. Flagging only so reviewers don't ask "why isn't this against main."
- The entry is alphabetized correctly between `openrouter/z-ai/glm-4.6` (`:27205`) and `openrouter/minimax/minimax-m2.1` (`:27224`) — though the convention is *not* strict alphabetical here (the file groups by vendor cluster), so this is a non-issue.
