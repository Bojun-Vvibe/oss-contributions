# Review: BerriAI/litellm #27249 — [Test] Stop parametrizing API keys into pytest test IDs

- Head SHA: `8743a84226f178439c62b9c213bdecbcb16aaf18`
- Files: `tests/audio_tests/test_audio_speech.py`, `tests/audio_tests/test_whisper.py`, `tests/local_testing/test_embedding.py` (and likely siblings, truncated)

## Summary

Refactors three test modules so that secret env-var values
(`AZURE_TTS_API_KEY`, `OPENAI_API_KEY`, `AZURE_WHISPER_API_KEY`,
`AZURE_AI_API_KEY`, plus their `_API_BASE` siblings) are no longer
substituted into pytest test IDs via `@pytest.mark.parametrize(...)`. The
fix pattern is consistent across all three files: the original parametrized
test body becomes a private `_run_<name>(...)` async helper, and one
`@pytest.mark.asyncio` test per provider literal-calls the helper with the
secret read inline (so the parametrize IDs become only the safe variants
like `sync_mode`, `response_format`).

## Specifics

- `tests/audio_tests/test_audio_speech.py:9-26` — original
  `@pytest.mark.parametrize("model, api_key, api_base", [(... os.getenv(...))])`
  block deleted; the parametrized two-cell matrix is replaced by a plain
  `_run_audio_speech_litellm(sync_mode, model, api_base, api_key)` helper.
- `tests/audio_tests/test_audio_speech.py:35-56` — two new tests
  `test_audio_speech_litellm_azure(sync_mode)` and
  `test_audio_speech_litellm_openai(sync_mode)` each parametrize *only*
  `sync_mode` and inline-read the secret env-vars. Pytest IDs become e.g.
  `test_audio_speech_litellm_azure[True]` — no key material.
- `tests/audio_tests/test_whisper.py:70-125` — same shape: original
  parametrize over `(model, api_key, api_base)` collapsed to
  `_run_transcription(...)` helper; two new tests
  `test_transcription_openai_whisper(...)` and
  `test_transcription_azure_whisper(...)` parametrize only the safe
  `(response_format, timestamp_granularities)` matrix.
- `tests/local_testing/test_embedding.py:193-200` (visible window) — the
  `azure_ai/Cohere-embed-v3-multilingual-2` parametrize entry that pulled
  `os.getenv("AZURE_AI_API_BASE")` and `os.getenv("AZURE_AI_API_KEY")` into
  the test ID is removed; refactor follows the same helper-based pattern
  (truncated at line 150 of 300 — confirm in full diff).

## Nits

- Genuine security win — pytest test IDs flow into CI logs, JUnit XML
  artifacts, and (most importantly) error tracebacks that are often pasted
  into issues. Previously a real key value would have been embedded in the
  test ID string and could leak into any of those surfaces. This refactor
  closes that. Worth a `SECURITY.md`/changelog mention with a CVE-style
  advisory if these test logs were ever publicly archived (CI artifacts on
  fork PRs in particular).
- The two new per-provider tests duplicate the parametrize decorator and
  the helper-call boilerplate — for the audio tests a single shared
  `_PROVIDERS = [("azure", ...), ("openai", ...)]` constant + a parametrize
  over an opaque key (`provider_id`) that the helper looks up internally
  would be more compact, but the current shape is clearer for grep so this
  is taste.
- No CI lint added to prevent regression. A simple
  `rg 'os\.getenv\([^)]+\).*pytest\.mark\.parametrize' tests/` guard or a
  custom pytest plugin that fails on parametrize args containing
  `os.getenv` calls would prevent the next contributor from re-introducing
  the same pattern.

## Verdict

`merge-after-nits`
