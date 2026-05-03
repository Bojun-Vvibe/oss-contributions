# BerriAI/litellm #27086 — fix: clear error for OpenRouter audio transcription (#27083)

- PR: https://github.com/BerriAI/litellm/pull/27086
- Author: kalp9197
- Head SHA: `713db73d741a5438d25d39cf1c2912f7f0db514f`
- Updated: 2026-05-03T13:54:58Z

## Summary
OpenRouter does not implement `/audio/transcriptions`, but `litellm.transcription(model="openrouter/...")` previously fell through into the OpenAI-compatible transcription path and produced a confusing 404 / generic error. This PR raises a clear `UnsupportedParamsError` at the top of `litellm.transcription()` when `custom_llm_provider == "openrouter"`, points the user at OpenAI + `whisper-1`, adds a README note, and ships sync + async tests.

## Observations
- `litellm/main.py:6490-6499`: the early-return guard is placed *after* `if dynamic_api_key is not None: api_key = dynamic_api_key` (line 6488) but *before* `get_optional_params_transcription`. That ordering is fine, but consider hoisting the guard above the dynamic-key resolution — there is no reason to look up an OpenRouter key just to immediately reject the call. Minor cleanup.
- `litellm/main.py:6491`: only the *prefix* form `custom_llm_provider == "openrouter"` is checked. If a user invokes via routed model strings like `openrouter/openai/whisper-1`, the provider extractor upstream of this function should already normalize `custom_llm_provider` to `"openrouter"`. Confirm with one extra test that calls `litellm.transcription(model="openrouter/openai/whisper-1", ...)` *without* explicit `custom_llm_provider=` actually trips this branch — the existing tests at `tests/test_litellm/llms/openai/transcriptions/test_openrouter_transcription_validation.py:11-19` do exactly that, good.
- `tests/.../test_openrouter_transcription_validation.py:32`: async test uses `pytest.mark.asyncio` but does not set `asyncio_mode`; if the project uses `asyncio_mode = "auto"` in `pytest.ini`, the marker is redundant; if not, this test relies on the marker being honored. Both are fine — just be consistent with the rest of the litellm test suite.
- No equivalent guard is added for sibling endpoints OpenRouter also lacks (e.g. `/audio/speech` TTS, `/images/*`, `/embeddings` for text-only routes that don't expose embeddings). This PR is narrowly scoped to transcription, which is fine — but file a follow-up issue or note in the PR body so the same UX gap on TTS doesn't surprise users.
- README addition at line 366 is a single line under the providers table; helpful but easy to miss. Consider a small "Provider gotchas" section if more such notes accrue.
- Error message uses `model=` and `llm_provider=` kwargs to `UnsupportedParamsError` — verify that error class signature matches (some LiteLLM exceptions take `model` positionally). The constructor at `litellm/exceptions.py` should be checked; a quick `pytest tests/test_litellm/llms/openai/transcriptions/test_openrouter_transcription_validation.py` run is enough to confirm.

## Verdict
`merge-after-nits`
