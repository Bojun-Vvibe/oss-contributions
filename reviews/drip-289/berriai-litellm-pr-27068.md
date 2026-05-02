# BerriAI/litellm PR #27068 — Preserve compaction iteration usage through OpenAI-shape conversion

- Head SHA: `107bade8b6023727f2da17de1576d778d9011bd7`
- URL: https://github.com/BerriAI/litellm/pull/27068
- Size: +67 / -0, 2 files (`litellm/llms/anthropic/chat/transformation.py`,
  `tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py`)
- Verdict: **merge-after-nits**

## What changes

Companion to drip-287's #27065 (which added `iterations` to the
Anthropic-shaped usage path). When Anthropic returns
`usage.iterations[]` for a compaction-firing request, the per-iteration
data is now carried through `AnthropicConfig.calculate_usage` into the
final OpenAI-shape `Usage` object so downstream cost / spend logic can
see *both* the deliberately-misleading top-level
`input_tokens`/`output_tokens` (Anthropic only counts non-compaction
iterations there) **and** the full iteration breakdown for accurate
billing.

`transformation.py:1789-1815` adds the `iterations` extraction and a
conditional `**({"iterations": iterations} if iterations is not None
else {})` splat into the `Usage(...)` constructor so existing callers
that don't know about the new field aren't disturbed.

## What looks good

- Test `test_calculate_usage_compaction_iterations`
  (test file:3661-3692) bakes in the *exact* Anthropic semantics
  the PR body links (top-level reflects only non-compaction; iterations
  array carries the truth). That comment on lines 3666-3669 is the
  most valuable thing in the PR — reviewers six months from now will
  not have to re-derive why the prompt_tokens=50 assertion makes sense
  given a 5000-token compaction iteration.
- The conditional splat (`transformation.py:1814` —
  `**({"iterations": iterations} if iterations is not None else {})`)
  is the right contract: the `Usage` schema can stay a `TypedDict`-like
  surface where unset fields are simply absent. Avoids polluting every
  non-Anthropic provider's serialized usage with a `"iterations": null`.
- `test_calculate_usage_no_iterations` (test file:3695-3705) asserts
  `not hasattr(usage, "iterations") or usage.get("iterations") is None`
  — the `or` covers both "real attribute" and "dict access" paths
  because `Usage` straddles both worlds. Defensive but correct.

## Nits

1. `transformation.py:1792-1794` only types `iterations` as
   `Optional[List[dict]]` and copies the raw list straight from the
   provider payload. Anthropic's iteration entries have a known shape
   (`iteration_type`, `input_tokens`, `output_tokens`, possibly
   `cache_creation_input_tokens`, `cache_read_input_tokens`) — a
   typed `List[AnthropicUsageIteration]` (TypedDict) would let mypy
   catch downstream consumers that index keys that aren't there.
2. The PR doesn't touch the cost-calculation layer that's *supposed*
   to consume `iterations`. The test proves the data round-trips into
   `Usage`, but if no caller in `litellm/cost_calculator.py` has been
   updated to actually sum `iteration["input_tokens"]` instead of
   trusting `usage.prompt_tokens`, this PR closes the data-plumbing
   half of issue #27060 but leaves the billing side underbilled. Worth
   either including the cost-calc patch or explicitly noting it's
   a follow-up.
3. The PR doesn't validate that `_usage["iterations"]` is actually a
   list (only checks `is not None`). A malformed provider response
   (`"iterations": {}`) will silently land a dict in `usage.iterations`
   and the consumer downstream will blow up far from the source. Add a
   `isinstance(_usage["iterations"], list)` guard.
4. No streaming-path coverage. Anthropic streams compaction events
   too; if the streaming aggregator builds its own `Usage` without
   going through `calculate_usage`, the streaming side will silently
   drop iterations. Worth at least an `xfail` test pinning the gap.

## Risk

Very low for the merge itself — the field is purely additive and
conditional. The risk is *informational*: anyone reading the PR title
"preserve compaction iteration usage" will assume billing is now
correct, but only the data-plumbing half is done. Update the PR
description to make that explicit, or land the cost-calculator patch
in the same PR.
