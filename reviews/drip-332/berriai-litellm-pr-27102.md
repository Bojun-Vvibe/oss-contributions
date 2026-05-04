# BerriAI/litellm PR #27102 — fix(deepseek): update reasoning_effort handling no thinking

- **Link:** https://github.com/BerriAI/litellm/pull/27102
- **Head SHA:** `e9740dc7f3632c41a2b043ec246f63115cb2f19b`
- **Author:** nqbao (Bao Nguyen)
- **Created:** 2026-05-04
- **Files:** `litellm/llms/deepseek/chat/transformation.py` (+11/−9), `tests/litellm/llms/deepseek/chat/test_deepseek_chat_transformation.py` (+17/−3)
- **Diff size:** +28 / −12

## Summary
Extends DeepSeek's `map_openai_params` so that:
1. `thinking={"type": "disabled"}` is now accepted (previously only `"enabled"` was passed through — `"disabled"` was silently dropped).
2. `reasoning_effort="none"` now produces `thinking={"type": "disabled"}` instead of being a no-op.

## Specific citations
- `litellm/llms/deepseek/chat/transformation.py:50-55` — accept-list expanded from `("enabled",)` to `("enabled", "disabled")`. The new code uses `thinking_value["type"]` directly instead of the hardcoded `"enabled"` literal — correct, because the input has already been validated against the tuple.
- `litellm/llms/deepseek/chat/transformation.py:59-63` — the `elif reasoning_effort is not None:` branch now bifurcates: `!= "none"` → enabled, `== "none"` → disabled.
- `tests/.../test_deepseek_chat_transformation.py:96-108` — old test `test_map_reasoning_effort_none_does_not_enable_thinking` is renamed and inverted to assert `result["thinking"] == {"type": "disabled"}`. **This is a behavior change**, not just a new feature — any existing caller that relied on `"none"` being a no-op will now see a `thinking` key materialize on the request.
- `tests/.../test_deepseek_chat_transformation.py:142-155` — new `test_map_thinking_disabled` verifies the explicit-disabled path.

## Observations
- DeepSeek's API does in fact accept `{"type": "disabled"}` (per their public docs as of late 2025) so the upstream contract is satisfied.
- The behavior flip on `reasoning_effort="none"` is the only thing I would flag as a release-note item. Most users will read "none" as "do not include reasoning", which is what `disabled` accomplishes — but a few may have been depending on the silent-drop path.
- Symmetric handling: `thinking_value=None` (not provided) still leaves `optional_params` untouched — correct.
- No change to the precedence test (`test_thinking_takes_precedence_over_reasoning_effort`), which is good and means the precedence rule is preserved.

## Nits
- The two-line `if reasoning_effort != "none": ... else: ...` could be a single dict lookup, but the explicit form reads fine and matches surrounding style.
- Add a CHANGELOG / release-notes entry calling out the behavior change for `reasoning_effort="none"` (was: dropped; now: emits `thinking={"type":"disabled"}`).
- Worth a one-liner test for `reasoning_effort=""` (empty string) to lock in the truthiness behavior — currently it would fall into the `!= "none"` branch and enable thinking, which is probably not desired.

## Verdict
`merge-after-nits`
