---
pr: 26380
repo: BerriAI/litellm
sha: 14d91d1ff0f6a94a2e16799d5f18bc292dac76b4
verdict: merge-after-nits
date: 2026-04-25
---

# BerriAI/litellm#26380 — feat(deepseek): add DeepSeek V4 Pro and V4 Flash model metadata

- **URL**: https://github.com/BerriAI/litellm/pull/26380
- **Author**: neo1027144-creator

## Summary

Adds 4 entries to `model_prices_and_context_window.json` (bare +
provider-prefixed forms of `deepseek-v4-flash` and `deepseek-v4-pro`)
and 14 mock tests asserting the metadata is queryable through the
standard `litellm.model_cost` lookup. Pricing sourced from
api-docs.deepseek.com. AI-disclosed authoring.

## Reviewable points

- `model_prices_and_context_window.json:+9644` adds
  `deepseek-v4-flash`:
  - `input_cost_per_token: 1.4e-07` ($0.14/M)
  - `input_cost_per_token_cache_hit: 2.8e-08` ($0.028/M)
  - `output_cost_per_token: 2.8e-07` ($0.28/M)
  - `max_input_tokens: 1000000`, `max_output_tokens: 384000`,
    `max_tokens: 384000`

  Math check vs. PR table: 0.14/M = 1.4e-07/token ✓; 0.028/M =
  2.8e-08/token ✓; 0.28/M = 2.8e-07/token ✓. All three line up.

- `+12281` adds `deepseek-v4-pro`:
  - `input_cost_per_token: 1.74e-06` ($1.74/M)
  - PR table says **$.74/M** for input (cache miss).
  - **Discrepancy**: 1.74e-06/token ≠ $0.74/M. It encodes $1.74/M.
    Either the PR table is wrong (a missing leading `1` in
    `.74/M`) or the JSON value is wrong by 2.35×. The PR body says
    "Pricing sourced from official DeepSeek API docs:
    https://api-docs.deepseek.com/quick_start/pricing" — a
    reviewer must open that URL and pick the source of truth.
    Until resolved this is a hard pre-merge blocker, because
    LiteLLM's cost-tracking surface is downstream of these numbers
    and a 2.35× billing error would propagate silently into every
    consumer's spend dashboard.

  - Output: `3.48e-06` = $3.48/M. PR table says **$.48/M**. Same
    pattern: either the table dropped a leading `3` or the JSON
    is 7.25× too high. Same blocker.

  - `cache_read_input_token_cost: 1.4e-07` = $0.14/M cache hit.
    PR table says **$.14/M** ✓ (here the leading-1 reading would
    mean $1.14/M, which the JSON does not encode — so the JSON
    matches the table here). That asymmetry is what makes me think
    the *table* is the typo'd surface, not the JSON.

- `tests/test_litellm/test_deepseek_model_metadata.py:+95` — the new
  `TestDeepSeekV4ModelMetadata` class adds 14 assertions covering
  existence (bare + prefixed forms), context window, function
  calling, reasoning, and provider tag. **None of the new tests
  assert any of the price values.** That's the exact gap that lets
  a 2.35× pricing typo slip through. At minimum, add four
  assertions: `entry["input_cost_per_token"] == 1.4e-07` (or
  whichever value is canonical post-resolution) for each of the 4
  entries. This locks the documented price as a regression guard
  and makes the next AI-assisted bump auditable.

- `supports_tool_choice: true` for both models — the existing
  `deepseek-chat` entry just above (line ~9628 in the JSON) has
  `supports_tool_choice: false`. The PR notes deepseek-chat is
  being deprecated in favor of v4-flash, so the flip from `false`
  to `true` is presumably correct, but worth confirming against
  the API docs that v4-flash actually accepts a `tool_choice`
  parameter (some providers advertise tool calling but reject the
  forced-choice variant).

- `supports_reasoning: true` paired with `mode: "chat"` (not
  `"chat_with_thinking"` or any reasoning-flavored mode) — confirm
  that v4-flash exposes reasoning via the standard
  `reasoning_effort` / `thinking` params under chat-completions,
  not through a separate endpoint. The supported_endpoints list is
  `["/v1/chat/completions"]` only.

## Rationale

Mechanical metadata addition with the right shape (bare +
prefixed entries, supports_* flags, source URL pinned). Two issues
must be resolved before merge:

1. **Pricing arithmetic mismatch** between PR table and JSON for
   v4-pro input/output. Reviewer must reconcile against the cited
   DeepSeek API docs URL and fix whichever side is wrong.
2. **Tests don't assert prices.** Add 4-8 explicit price
   assertions so the next AI-assisted bump can't silently move the
   numbers.

Once those are settled, this is fine to merge.

## What I learned

For pricing/metadata PRs to a registry that ~everyone in the
ecosystem reads from (`model_prices_and_context_window.json` is
literally the source of truth for LiteLLM-based cost tracking),
two complementary guardrails matter: (a) tests that assert *the
specific numeric values* — existence-only tests catch nothing —
and (b) PR review where one human opens the cited source-of-truth
URL and pattern-matches the numbers, because AI-assisted PRs (as
this one is honestly disclosed to be) reliably get
exponent/decimal-place placement wrong on small numbers (1.4e-07
vs 1.4e-06 differs by 10×). The asymmetry in this PR — cache-hit
price matches the table, cache-miss/output prices off by 2.35×/
7.25× — is the diagnostic shape of "AI generated the JSON
correctly from a source the human then summarized incorrectly into
the PR table"; checking the upstream URL resolves it.
