---
pr: BerriAI/litellm#26800
sha: 3f5c58925571381af50996b54ca87bf53ebd3180
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(bedrock): add 1-hour cache write pricing for Claude 4.5/4.6/4.7 (Global, US)

URL: https://github.com/BerriAI/litellm/pull/26800
Files: `litellm/model_prices_and_context_window_backup.json`, `model_prices_and_context_window.json`
Diff: 167+/0-

## Context

Anthropic offers two prompt-cache TTL tiers on Bedrock: 5-minute (default,
already priced in litellm via `cache_creation_input_token_cost`) and 1-hour
extended cache (priced ~1.6× the 5-minute write rate). Until this PR, every
Claude 4.5 / 4.6 / 4.7 entry in the litellm price catalog was missing the
1-hour cache write key, so any caller using `cache_control: {ttl: "1h"}`
got billed at the 5-minute rate in litellm's spend logs while AWS billed at
the 1-hour rate — silent under-counting in the cost-tracking layer.

## What's good

- Adds `cache_creation_input_token_cost_above_1hr` to the right entries
  uniformly across both the active catalog (`model_prices_and_context_window.json`)
  and its backup mirror (`...backup.json`). The 14 model variants touched
  cover Haiku 4.5, Sonnet 4.5, Sonnet 4.6, Opus 4.5, Opus 4.6, Opus 4.7
  in the `anthropic.*`, `global.anthropic.*`, and `us.anthropic.*`
  region prefixes — that's the full Bedrock matrix, not a partial fix.
- Pricing math checks: e.g. Sonnet 4.5 base
  `cache_creation_input_token_cost: 3.75e-06` → `_above_1hr: 6e-06`
  (1.6× ratio). Haiku 4.5 `1.25e-06` → `2e-06` (1.6×). Opus 4.5/4.6/4.7
  `6.25e-06` → `1e-05` (1.6×). All consistent with Anthropic's published
  multiplier (and the US-region 10% Bedrock surcharge correctly carried
  through: us.opus 4.5 `6.875e-06` → `1.1e-05`, us.haiku 4.5 `1.375e-06` →
  `2.2e-06`).
- For the long-context Sonnet 4.5 entries that already had
  `cache_creation_input_token_cost_above_200k_tokens`
  (`...backup.json:1459-1467`), the PR also adds the matching
  `cache_creation_input_token_cost_above_1hr_above_200k_tokens` so the
  `(ttl=1hr) × (above_200k)` quadrant is correctly priced — easy to forget,
  caught here.
- Both files updated atomically — no skew between the live and backup
  catalog that would cause non-deterministic pricing depending on which
  loader path runs.

## Verdict reasoning

Pure pricing-data PR with mechanical, verifiable additions. No code paths
touched; downstream consumers only get a richer cache TTL pricing model.
The kind of fix that should land same-day to stop spend-log under-reporting
on production accounts using extended caching.
