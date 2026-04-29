# BerriAI/litellm PR #26732 — fix(model-prices): add jp.anthropic.claude-{sonnet-4-6,opus-4-7} Bedrock entries

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26732
- Head SHA: `bdcdcc7c36`
- State: OPEN, +58/-0 across 1 file (fixes #22972)

## What it does

Adds two missing Bedrock-Converse pricing entries to `model_prices_and_context_window.json`: the Japan (`jp.`) cross-region inference profiles for `anthropic.claude-opus-4-7` and `anthropic.claude-sonnet-4-6`. These are AWS regional alias keys for the same underlying model, with **JP-region pricing premium** baked in (input/output costs are higher than the un-prefixed base model entries).

## Specific reads

- `model_prices_and_context_window.json:1290-1322` — new `jp.anthropic.claude-opus-4-7` block:
  - `input_cost_per_token: 5.5e-06` vs base `anthropic.claude-opus-4-7` which a quick grep above this block shows at lower per-token rates — JP premium of roughly ~10-15% on input is consistent with AWS published Bedrock JP pricing for Anthropic models. Sanity-check by comparing with the existing `jp.` entries for older claude models to confirm the proportional premium is in the same ballpark.
  - `cache_creation_input_token_cost: 6.875e-06` (= 1.25× input cost — matches Anthropic's standard cache-creation multiplier).
  - `cache_read_input_token_cost: 5.5e-07` (= 0.1× input cost — matches Anthropic's standard cache-read multiplier).
  - `output_cost_per_token: 2.75e-05` (= 5× input cost — matches Anthropic's standard output multiplier).
  - All four ratios are internally consistent with each other, which suggests the input cost was set first and the rest derived correctly.
  - `max_input_tokens: 1000000`, `max_output_tokens: 128000`, `max_tokens: 128000`. Matches base opus-4-7.
  - `tool_use_system_prompt_tokens: 346` — same as base entry. ✓
  - Capability flags: `supports_computer_use`, `supports_function_calling`, `supports_pdf_input`, `supports_prompt_caching`, `supports_reasoning`, `supports_response_schema`, `supports_tool_choice`, `supports_vision`, `supports_xhigh_reasoning_effort`, `supports_native_structured_output`, `supports_max_reasoning_effort`, `supports_minimal_reasoning_effort`. Notably `supports_assistant_prefill: false` (consistent with what I'd expect for Bedrock-Converse vs direct Anthropic API). All flags match the base `anthropic.claude-opus-4-7` block.
  - `search_context_cost_per_query: { high: 0.01, low: 0.01, medium: 0.01 }` — flat $0.01 across tiers. Same as base entry.
- `model_prices_and_context_window.json:1462-1490` — new `jp.anthropic.claude-sonnet-4-6` block:
  - `input_cost_per_token: 3.3e-06`, `output_cost_per_token: 1.65e-05`, `cache_creation: 4.125e-06`, `cache_read: 3.3e-07`. Same 1.25×/0.1×/5× ratios as opus block. ✓
  - `max_input_tokens: 1000000`, `max_output_tokens: 64000` (vs opus's 128000) — matches base sonnet-4-6 entry.
  - `supports_assistant_prefill: true` (vs opus's `false`) — consistent with sonnet's API capability shape. Worth flagging this asymmetry to a reviewer who might assume the JP entries should mirror each other; they shouldn't, they should mirror their respective base entries.
  - **Missing capability flags vs opus block**: `supports_xhigh_reasoning_effort` and `supports_max_reasoning_effort` are absent. Compare with the base `anthropic.claude-sonnet-4-6` entry (just above in the file) — does the base also omit these? If the base has them and the `jp.` entry omits them, that's a copy-paste bug. If the base also omits them (sonnet-4-6 doesn't support those reasoning levels), then the `jp.` mirror is correct. Quick verification against `anthropic.claude-sonnet-4-6:1290-` would settle it. Without that confirmation here, flag for human review.
- **PR template compliance**: description says "Adding at least 1 test is a hard requirement." Pricing-data PRs typically pass with a `tests/test_litellm/test_model_prices.py`-style schema-validation test (or a regional-pricing-parity test). The diff doesn't include one. Either there's an automated JSON-schema-validation test that runs on every change to this file (likely — LiteLLM has one), or the PR needs a unit test asserting `litellm.model_cost["jp.anthropic.claude-opus-4-7"]["input_cost_per_token"] == 5.5e-06`. Either way the maintainer policy may want a one-line test added explicitly.
- **Source-of-truth**: the PR description doesn't link to the AWS Bedrock pricing page that was used to derive these numbers. For pricing PRs, reviewers (and future auditors who hit a "did the rates change?" bug report) need a citation. Recommend adding the AWS pricing URL or a screenshot.

## Risk

1. **Pricing accuracy** is the only real risk class for this PR. Wrong rates → wrong cost-tracking → wrong customer invoicing for any LiteLLM operator routing to JP-region Anthropic-on-Bedrock. The four-ratio internal consistency check (input/cache-create/cache-read/output ratios) gives strong evidence the numbers were derived consistently, but doesn't prove the *base* input rate is correct. Cross-check against AWS Bedrock pricing for ap-northeast-1.
2. **Capability-flag drift** between the opus and sonnet JP entries (specifically the missing `supports_xhigh_reasoning_effort` / `supports_max_reasoning_effort` on sonnet) is most likely correct (mirrors base sonnet) but should be explicitly checked.
3. **Key-naming consistency**: this PR uses `jp.anthropic.X` prefix. Verify that other regional-prefixed entries in the file use the same `<region>.anthropic.<model>` shape (e.g. is there `eu.anthropic.X`? `apac.anthropic.X`? `us.anthropic.X`?). Inconsistent prefix-style across regions is a foot-gun for routing config.
4. No risk to existing entries — purely additive.

## Verdict

`merge-after-nits` — additive pricing data with internally-consistent ratios. Two pre-merge nits: (a) link AWS Bedrock JP pricing as the source of truth in PR description, (b) confirm the capability-flag asymmetry between opus and sonnet matches their respective base entries (a 30-second `jq` diff would do it). One post-merge nit: ensure the test policy is satisfied (add a presence-test or trust the existing JSON-schema validation).

## What I learned

Pricing-only PRs to centralized rate tables are deceptively risky — the diff looks trivial but the blast radius is "every operator's billing reconciliation breaks if the numbers are wrong." The right reviewer reflex is (1) verify ratio-consistency across cache/output multipliers (catches arithmetic errors), (2) verify capability-flag parity with the base entry (catches copy-paste-from-wrong-source errors), (3) verify against the upstream pricing page (catches stale-data errors), and (4) confirm key-naming convention matches sibling regional entries (catches lookup-key drift that surfaces months later as a "model not found" routing bug).
