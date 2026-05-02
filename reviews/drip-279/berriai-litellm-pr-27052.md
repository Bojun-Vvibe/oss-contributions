# BerriAI/litellm PR #27052 — Add Azure AI Kimi K2.6 metadata

- URL: https://github.com/BerriAI/litellm/pull/27052
- Head SHA: `92ffb55fe97303cf94f95bd9644bb05a2da346a1`
- Files touched: 2 (additions only)
- Verdict: **merge-as-is**

## Summary
Adds `azure_ai/kimi-k2.6` model entries to both
`model_prices_and_context_window.json` and the `_backup` copy. Pricing,
context window, modalities, and capability flags are populated.

## Cited concerns

1. `litellm/model_prices_and_context_window_backup.json:6893-6911` and
   `model_prices_and_context_window.json:6907-6925`: identical entries written
   to both files, which matches the established convention in this repo
   (the `_backup` is a snapshot the runtime falls back to). Numbers match:
   `input_cost_per_token: 9.5e-07`, `output_cost_per_token: 4e-06`,
   `max_input_tokens: 262144`, `max_output_tokens: 262144`. Consistent.

2. The `source` URL points to a public Azure AI Foundry blog post — that's
   the canonical reference this repo prefers for `azure_ai/*` rows, so the
   pricing is auditable.

3. Capability flags include `supports_reasoning: true` but the entry omits a
   `supports_reasoning_effort` flag (some other reasoning-capable rows in the
   file include it). Worth confirming with upstream Azure docs whether K2.6
   accepts an `effort` parameter; if so, a follow-up can add the flag plus
   the supported effort tiers. Not a blocker — `supports_reasoning` is the
   primary contract the proxy router checks.

4. Pure data PR; no code paths exercised, no tests required by repo
   convention.

## Verdict
**merge-as-is** — clean metadata addition that matches existing schema and
sourcing conventions. Capability-flag follow-up can happen in a separate PR
once the reasoning-effort behavior is confirmed against the live endpoint.
