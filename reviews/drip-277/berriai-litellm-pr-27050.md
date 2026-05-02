# Review — BerriAI/litellm#27050

- PR: https://github.com/BerriAI/litellm/pull/27050
- Title: fix(reasoning_effort): raise BadRequestError(400) on invalid effort values
- Head SHA: `03ea1f83b3a519a036e49b58eba303fee407da02`
- Size: +135 / −15 across 5 files
- Verdict: **merge-after-nits**

## Summary

Three `raise ValueError(...)` sites in
`litellm/llms/anthropic/chat/transformation.py` are converted to
`raise litellm.BadRequestError(...)`. Previously, an invalid
`reasoning_effort` (e.g. `"disabled"`), an invalid `output_config.effort`,
or `effort="max"` / `"xhigh"` on an unsupported model would surface
through the proxy as a 500 APIConnectionError because litellm wraps
unhandled `ValueError` as a generic upstream failure. With the change
the user gets a clean HTTP 400 with a structured message naming the
allowed values. Bedrock and Databricks call sites that delegate to
`AnthropicConfig._map_reasoning_effort` are updated to forward
`llm_provider="bedrock"` / `"databricks"` so the BadRequestError carries
the correct provider attribution.

## Evidence

- `litellm/llms/anthropic/chat/transformation.py:794-840` — the
  `_map_reasoning_effort` signature gains
  `llm_provider: str = "anthropic"`, and the terminal `else` arm
  raises `litellm.BadRequestError(message=..., model=model,
  llm_provider=llm_provider)` instead of `ValueError`.
- Same file, lines 1090-1104: the `map_openai_params` call site now
  passes `llm_provider=self.custom_llm_provider or "anthropic"` so a
  Bedrock-routed request gets `llm_provider="bedrock"` on the
  resulting BadRequestError.
- Lines 1539-1577 (`_apply_output_config`): three `raise ValueError`
  sites for invalid `effort`, unsupported `effort="max"`, and
  unsupported `effort="xhigh"` all become `raise
  litellm.BadRequestError(...)` with the same provider plumbing.
- `litellm/llms/bedrock/chat/converse_transformation.py:452-456` and
  `litellm/llms/databricks/chat/transformation.py:333-337` pass
  `llm_provider="bedrock"` / `"databricks"` into the helper so the
  resulting 400 isn't mis-attributed to anthropic.
- Tests: `tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py`
  updates the existing `test_effort_validation` /
  `test_max_effort_rejected_for_*` cases to expect
  `litellm.BadRequestError` instead of `ValueError`, and adds three
  new parametrised cases (`test_invalid_reasoning_effort_raises_bad_request`,
  `test_invalid_output_config_effort_raises_bad_request`,
  `test_unsupported_xhigh_raises_bad_request`) that explicitly
  assert `exc_info.value.status_code == 400`.

## Notes / nits

- `_map_reasoning_effort` is a `@staticmethod`-style helper invoked
  from at least three providers (anthropic, bedrock, databricks)
  plus the `map_openai_params` site. Defaulting `llm_provider` to
  `"anthropic"` is pragmatic, but a missed call site will silently
  mis-attribute the error. A follow-up that drops the default and
  forces every caller to pass it explicitly would catch any future
  regressions at compile time (in Python: at import-time linter
  warnings via mypy/ruff).
- The `f"...{reasoning_effort!r}..."` formatting will produce
  `'disabled'` (with quotes) in the message, which is good for the
  empty-string case (`''`) but slightly noisy for normal values.
  Acceptable.
- One concern: any external code that currently catches
  `ValueError` from these helpers (custom routers / pre-call hooks)
  will silently stop catching after this lands. Worth a one-line
  changelog entry calling out the exception-type change so
  downstream proxy operators don't get surprised.

Ship after the changelog note.
