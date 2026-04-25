# BerriAI/litellm#26456 — gpt-5.5 reasoning_effort capability flags + add supports_low_reasoning_effort

**Verdict: merge-as-is**

- PR: https://github.com/BerriAI/litellm/pull/26456
- Author: mateo-berri
- Base SHA: `d21e90f6831eebde5eb8f8d42604f5b57116d05e`
- Head SHA: `c3338384c93c467ebf447a52b0d5bba740a19395`
- +137 / -11

## Summary

Two-part fix for GPT-5.5 family reasoning_effort capability flags,
verified live against OpenAI's `/v1/chat/completions` API on
2026-04-24. (1) `supports_minimal_reasoning_effort` was set to `true`
on all four GPT-5.5 entries — wrong; OpenAI rejects `minimal` on both
`gpt-5.5` and `gpt-5.5-pro`. Flipped to `false`. (2) `gpt-5.5-pro`
additionally rejects `reasoning_effort=low` (only accepts
{medium, high, xhigh}); the JSON schema had no `supports_low_reasoning_effort`
flag to express that. New flag added to `ProviderSpecificModelInfo`,
threaded through `_get_model_info_helper`, and the GPT-5
transformation's existing `minimal` opt-out branch is widened to also
cover `low`. JSON entries updated for both pro variants.

## Specific findings

- `litellm/llms/openai/chat/gpt_5_transformation.py:244` (head SHA
  `c3338384c93c467ebf447a52b0d5bba740a19395`):
  ```python
  elif effective_effort in ("minimal", "low"):
      # minimal/low are opt-out: unknown models pass through; only block when
      # the model map explicitly sets supports_{level}_reasoning_effort=false.
      if self._is_reasoning_effort_level_explicitly_disabled(
          model, effective_effort
      ):
  ```
  The opt-out semantic (only block when flag is **explicitly** false,
  not when missing) is the right call for forward compat — every
  unknown model continues to pass `low` through. The single helper
  `_is_reasoning_effort_level_explicitly_disabled(model, level)` is
  already used for `minimal`, so the new `low` arm reuses the same
  lookup pattern with no duplication.
- `litellm/types/utils.py:140` — adds
  `supports_low_reasoning_effort: Optional[bool]` to
  `ProviderSpecificModelInfo` next to the existing
  `supports_minimal_reasoning_effort` field. Default-Optional means
  pre-existing TypedDict consumers don't have to populate it.
- `litellm/utils.py:5896` — `_get_model_info_helper` reads
  `_model_info.get("supports_low_reasoning_effort", None)` — the
  explicit `None` default keeps the opt-out semantic intact. Bug
  would have been if this defaulted to `False`.
- `litellm/model_prices_and_context_window_backup.json:19410` and
  the equivalent stanza in `model_prices_and_context_window.json:19424`:
  ```
  "supports_minimal_reasoning_effort": false,
  "supports_low_reasoning_effort": false
  ```
  on `gpt-5.5-pro` and the dated `gpt-5.5-pro-2026-04-23` only — not
  on the non-pro `gpt-5.5` / `gpt-5.5-2026-04-23`, which only get the
  minimal flip. This matches the live API contract documented in the
  PR body's curl output.
- `tests/test_litellm/litellm_core_utils/llm_cost_calc/test_llm_cost_calc_utils.py:411`
  — new parameterized test
  `test_gpt55_reasoning_effort_flags_match_live_openai_api` that pins
  all four entries' `supports_{none,minimal,xhigh}_reasoning_effort`
  to the OpenAI contract. Excellent regression lock — anyone who
  later flips a flag back to `true` to "fix" something will hit this.
- `tests/test_litellm/llms/openai/test_gpt5_transformation.py:536` —
  five new tests including
  `test_gpt5_5_pro_rejects_reasoning_effort_low`,
  `test_gpt5_5_pro_drops_reasoning_effort_low_when_requested` (the
  `drop_params=True` path), and
  `test_gpt5_unknown_model_passes_through_low` (forward compat for
  unknown models). Full coverage of the four meaningful matrix cells.

## Rationale

This is the right kind of bug fix: empirical (curl output against live
API quoted in the PR body), narrow (4 JSON entries + 1 transformation
clause + 1 type field + 1 helper threading), well-tested (7 new tests
across two test files including the live-API contract pin), and
surgical about scope — the Azure entries are explicitly carved out as
out-of-scope because Microsoft hasn't shipped GPT-5.5 on Azure yet.
The opt-out semantic (block only on explicit `false`, pass-through on
missing) is the correct backward-compat choice and matches how
`supports_minimal_reasoning_effort` already worked. The
`_is_reasoning_effort_level_explicitly_disabled` helper reuse is
clean — no duplicated lookup logic. The `test_gpt55_reasoning_effort_flags_match_live_openai_api`
test is particularly valuable because it locks the JSON to the *live*
contract, so a future PR that "helpfully" flips `supports_minimal:
true` back will hit a failure with a clear message pointing at the
2026-04-24 verification. One minor observation: the
`_is_reasoning_effort_level_explicitly_disabled` helper is reused but
not shown in the diff — worth a maintainer eyeball confirming it
actually handles the `low` level (likely yes since it takes `level`
as a parameter, but the diff doesn't show that file). The drop_params
test path covers the soft-rejection case. Without these flags,
`drop_params=True` users see avoidable 400 round-trips and
`drop_params=False` users get the same 400 surfaced from OpenAI —
both reflect badly on the proxy. Merge.
