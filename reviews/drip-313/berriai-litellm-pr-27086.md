# BerriAI/litellm PR #27086 — fix: clear OpenRouter audio transcription error

- Author: kalp9197
- Head SHA: `037764099dfb29f1066bc4e02fb868544ed9f5c1`
- Verdict: **merge-after-nits**

## Rationale

Adds an early-exit guard so OpenRouter audio-transcription requests fail
with `UnsupportedParamsError` instead of a confusing downstream error. New
helper at `litellm/llms/openrouter/transcription_validation.py:6-22` is a
3-line check, called from `litellm/main.py:6493-6496` right after
`get_llm_provider` resolution. Tests at
`tests/test_litellm/llms/openai/transcriptions/test_openrouter_transcription_validation.py:13-58`
cover sync, async, no-op for non-openrouter providers, and direct helper
invocation. README note at `README.md:368` is fine. Two nits: (1) test
file lives under `openai/transcriptions/` but tests OpenRouter — move to
`openrouter/` for consistency; (2) `model="openrouter/openai/whisper-1"`
in the test can mislead readers — pick a model id that doesn't pretend
OpenRouter has a whisper alias. Otherwise small, well-tested, and the
error message is genuinely better than the current behavior.
