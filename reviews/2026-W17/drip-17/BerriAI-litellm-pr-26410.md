# BerriAI/litellm PR #26410 — fix: add vertex sonnet 4.6 1h cache pricing

- **Author:** biubiubiuboomboomboom
- **Head SHA:** 313686b1fefbf689d98ea78c6b8b0c1c4b698e13
- **Files:** `model_prices_and_context_window.json` (+2 / −0)
- **Verdict:** `merge-as-is`

## Context

Anthropic's prompt-caching has two TTL tiers: a 5-minute ephemeral
cache (the default `cache_creation_input_token_cost`) and a 1-hour
extended cache, billed at a higher write cost. Vertex's hosted
`claude-sonnet-4-6` exposes both, but the litellm pricing table only
has the 5-minute write cost. Calls that opt into the 1h tier are
under-billed in cost reporting / budget enforcement.

## What the diff does

Two surgical insertions into `model_prices_and_context_window.json`:

- Line 31510 (`vertex_ai/claude-sonnet-4-6` entry):
  ```json
  "cache_creation_input_token_cost_above_1hr": 6e-06,
  ```
- Line 38405 (`vertex_ai/claude-sonnet-4-6@default` alias entry):
  ```json
  "cache_creation_input_token_cost_above_1hr": 6e-06,
  ```

Both alongside the existing `cache_creation_input_token_cost: 3.75e-06`.

## Review notes

- The numbers look right: $3.75/MTok 5-min write vs $6.00/MTok 1h
  write is consistent with Anthropic's published 1.6× premium for
  the extended cache (3.75 × 1.6 = 6.00). `cache_read_input_token_cost`
  stays at 3e-07 because reads are billed once regardless of TTL.
- The `cache_creation_input_token_cost_above_1hr` key is the existing
  litellm convention — already present on direct Anthropic
  `claude-sonnet-*` entries — so cost-tracking and budget code
  paths will pick this up automatically without any callsite
  changes. That's the right reason to do this as a JSON-only PR.
- Both the canonical name and `@default` alias are updated, which
  matches the pattern elsewhere in this file (the `@default` alias
  is what the model-router resolves to when the user omits a
  version pin). Forgetting one would mean half of users get the
  wrong cost. Good catch.
- No JSON-schema changes required: `model_prices_and_context_window.json`
  is a free-form dict of model → metadata, and the
  `*_above_1hr` key is already documented.
- No test added, which is fine — this file is data, and litellm's
  cost-calculation tests already cover the
  `cache_creation_input_token_cost_above_1hr` code path on direct
  Anthropic entries. Adding a regression assertion that vertex SKUs
  match the equivalent direct-Anthropic SKU would be nice but is
  out of scope for a 2-line pricing fix.
- Verify against Vertex's actual pricing page before merge — that's
  the only source of truth; the diff cites no source URL in the
  PR body. A one-line link in the commit message would harden
  this against future "wait, where did this number come from?"
  archaeology.

## What I learned

When an upstream provider exposes a feature flag (here: 1h cache
TTL) that the SDK already understands at the per-model-family level,
ratifying it for new model SKUs is a 2-line data change rather than
a code change. The risk surface is just "is the number right" and
"did you remember every alias of the model name" — the latter is
the bug class most likely to recur, and it's caught here.
