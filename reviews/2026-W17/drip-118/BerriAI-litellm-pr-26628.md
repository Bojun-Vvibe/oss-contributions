# PR #26628 — Fix unsupported thinking_budget=0 for Gemini Pro

- **Repo**: BerriAI/litellm
- **PR**: #26628 (DRAFT)
- **Head SHA**: `d22ffd94`
- **Author**: musse (Felipe Musse)
- **Size**: +107 / -1 across 2 files
- **Verdict**: **merge-as-is**

## Summary

Real, well-diagnosed router-fallback bug. When the
`gemini-2.5-flash` deployment hits 429 RESOURCE_EXHAUSTED and
the router falls back to `gemini-2.5-pro` with the same
translated request, LiteLLM was forwarding `thinkingBudget: 0`
into the Pro request body. Pro rejects that with
`400 INVALID_ARGUMENT — The model does not support setting
thinking_budget to 0`, which torches the fallback and surfaces
to the user as a hard error even though both deployments are
notionally healthy. The fix is a 4-line carve-out in
`_map_thinking_param` that omits `thinkingBudget` from the
request body when the value is `0` and the model is not
Flash-family. Non-zero budgets and Flash models are
unaffected.

## Specific changes

- `litellm/llms/vertex_ai/gemini/vertex_and_google_ai_studio_gemini.py:925-931` — new static helper `_model_supports_thinking_budget_zero(model: Optional[str]) -> bool` that returns `"flash" in model.lower()` (with a `None`-guard returning `False`). Docstring captures the API contract: "Only Flash-family Gemini 2.x models accept thinkingBudget=0 to disable thinking. Pro models reject it with a 400 INVALID_ARGUMENT error."
- `litellm/llms/vertex_ai/gemini/vertex_and_google_ai_studio_gemini.py:1015-1020` — gates the existing `params["thinkingBudget"] = thinking_budget` write behind `if thinking_budget > 0 or VertexGeminiConfig._model_supports_thinking_budget_zero(model)`. Order is right: positive budgets always pass through (the previous behavior), and `0` only passes through for Flash-family. The carve-out is on the boolean OR side, not a separate `if/else` branch — keeps the diff readable.
- `tests/llm_translation/test_gemini.py:1613-1706` — new regression test `test_gemini_flash_to_pro_fallback_thinking_budget_zero` with three layered assertions: (a) translation-layer call against `_map_thinking_param` directly asserts `"thinkingBudget" not in pro_config` for Pro and `flash_config.get("thinkingBudget") == 0` for Flash; (b) Router-layer test with `mock_acompletion` patched to raise `RateLimitError` for flash and capture the kwargs sent to pro, asserting `call_count == 2` and the response content roundtrips; (c) the captured `pro_call_kwargs.get("thinking")` still equals the original `{"type": "enabled", "budget_tokens": 0}` — meaning the user-supplied param is preserved end-to-end and only the *translated* `thinkingBudget` is stripped. This is the right granularity: pin both the transformer and the integration outcome.

## Risks

- **`"flash" in model.lower()` is a substring check**: would also match a future `gemini-3-flash-pro` (hypothetical), or a custom router model name like `my-flash-tier`. Probably fine for the Gemini family today since Google's naming convention has been stable, but a regex `r"\bflash\b"` or an enum-style allow-list would scale better. Worth a one-line note.
- **The PR's pre-submission checklist has the Greptile review checkbox unchecked**: BerriAI has flagged that they want a Greptile confidence ≥4/5 before maintainer review. The PR is in DRAFT state which is consistent with that, but the comments thread shows greptile-apps already commented — the author should confirm they got their score.
- **Test relies on `litellm.acompletion` being patchable at module level**: the `with patch("litellm.acompletion", side_effect=mock_acompletion)` works because Router calls back into the top-level `acompletion` symbol, but if the test environment ever isolates that import differently (e.g. `from litellm.main import acompletion as acompletion`), the patch silently targets the wrong symbol and the test passes for the wrong reason. A `mock_acompletion.assert_called_with(...)` on the Pro call to confirm it actually fired would catch that.
- **No coverage for `model=None` path**: the helper handles `None` but the test doesn't exercise it. Cheap to add.
- **Bare `if "flash" in model.lower()` at module scope means a future Pro variant that happens to want this carve-out also has to thread the model name — not a fault of this PR**, but it's worth landing a follow-up that turns this into a per-model capability table (`MODEL_SUPPORTS_THINKING_BUDGET_ZERO: set[str]`) so the next "Pro 3.0 supports it" tweak is a one-line config change.

## Verdict

`merge-as-is` — this is the textbook shape of a "router fallback
crashes because translated request shape isn't portable across
sibling models" bug. The diagnosis is correct, the fix is
minimal and applied at the right layer (the per-model
translator, not the router), and the test pins both the
transformer behavior in isolation and the end-to-end fallback
path that motivated the fix. The Greptile checkbox should be
confirmed but the substantive code is ready.

## What I learned

This is a clean illustration of why router-level fallback needs
either (a) per-model request translation that's idempotent
under model swap (what this PR moves toward) or (b) a
"re-translate on fallback" hook that re-runs the model-specific
transform when the model changes mid-call. LiteLLM does the
former, which is the right choice for the steady-state path,
but it puts the burden on every translator to know about every
sibling-model quirk. A capability-table approach (each model
declares which thinking-budget values it accepts) would be more
scalable than substring-matching the model name, and the
"Flash supports 0, Pro doesn't" fact would live in one place
instead of being inlined into a translator that already does
six other things.
