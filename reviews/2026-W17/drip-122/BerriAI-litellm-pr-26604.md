---
pr: 26604
repo: BerriAI/litellm
sha: cfe0b774aeaaebe3f9b0cb0760fd3256e47e23d6
verdict: merge-as-is
date: 2026-04-28
---

# BerriAI/litellm #26604 — fix(health-check): strip max_tokens before non-chat handlers

- **Author**: xr843 (Tim Ren)
- **Head SHA**: `cfe0b774aeaaebe3f9b0cb0760fd3256e47e23d6`
- **Size**: 21 added / 2 deleted in 1 file
  (`litellm/litellm_core_utils/health_check_utils.py`).
- **Closes**: #26406

## Scope

Closes a permanent-unhealthy-status bug for `image_generation`,
`video_generation`, `embedding`, `rerank`, `audio_speech`,
`audio_transcription`, `ocr`, `responses`, and `batch` deployments in the
proxy `/health` UI. Root cause: `_update_litellm_params_for_health_check`
unconditionally injects both `messages` and `max_tokens` into
`litellm_params` for every deployment; `_filter_model_params` was only
stripping `messages`, so `max_tokens` flowed through to handlers that don't
accept it. OpenAI's image-generation endpoints (`dall-e-2`, `dall-e-3`,
`gpt-image-1`) are strict about unknown parameters and respond with
`400 BadRequestError: Unknown parameter: 'max_tokens'`. Other providers
(Vertex Imagen, Bedrock) silently drop unknown fields, masking the bug
to operators on those backends.

## Specific findings

- `litellm/litellm_core_utils/health_check_utils.py:5-12` — new
  `_NON_CHAT_HEALTH_CHECK_STRIP_KEYS = {"messages", "max_tokens"}`
  module-level constant. Module-level (good — discoverable via grep,
  not rebuilt per call), `set` (good — `in`-membership is the only
  use), and the comment correctly names the failure mode (OpenAI
  image generation 400) plus the rationale (these fields are
  chat/completion-only). The constant name matches its usage:
  "things to strip when routing through a non-chat handler."
- `litellm/litellm_core_utils/health_check_utils.py:15-27` — rewritten
  `_filter_model_params`:
  ```python
  return {
      k: v
      for k, v in model_params.items()
      if k not in _NON_CHAT_HEALTH_CHECK_STRIP_KEYS
  }
  ```
  Replaces the previous one-line `if k != "messages"` filter with the
  set membership. Correct shape — switches from per-key special-case to
  set-based contract that's easy to extend (next chat-only injection that
  needs the same treatment is a one-line addition to the constant).
- `litellm/litellm_core_utils/health_check_utils.py:18-23` — the
  docstring honestly names the load-bearing invariant: "`litellm.acompletion`
  is the only mode handler that consumes `model_params` unfiltered; every
  other handler routes through this helper, so removing chat-completion-only
  keys here keeps strict providers (OpenAI image generation, etc.) from
  rejecting the request." This is exactly the right framing — the change
  is safe precisely because chat is the only path that bypasses
  `_filter_model_params`, so chat health checks continue to receive
  `max_tokens` for cost control.

## Risk

Low. The change is strictly subtractive (remove `max_tokens` from the
dict that flows into non-chat handlers) and the docstring correctly
identifies the one consumer (`litellm.acompletion`) that bypasses this
helper, so chat health checks are unaffected by definition. The
`BACKGROUND_HEALTH_CHECK_MAX_TOKENS` env var continues to apply to chat
deployments because that knob feeds into the unfiltered chat path.

The only thing this change cannot fix is non-chat handlers that *also*
require chat-only fields not currently in the strip set — but the
documented failure (OpenAI image-gen 400 on `max_tokens`) is the one
the PR claims to fix, and the constant-set shape makes future additions
trivial.

The PR's test plan is honest: "Module parses; inline check confirms
`_filter_model_params({"model": "openai/dall-e-3", "messages": [...],
"max_tokens": 50, "api_key": "..."})` returns `{"model": ..., "api_key":
...}` without `messages` or `max_tokens`." — this is the right minimal
assertion. A unit test pinning the same shape inside the test suite
would be a nice-to-have but the existing health-check test surface
likely covers the indirect path; this PR shouldn't be blocked on adding
one for a 2-line behavior change.

## Verdict

**merge-as-is** — surgical fix at the exact filter boundary, module-level
constant for extensibility, docstring naming the load-bearing invariant
("acompletion is the only unfiltered consumer"). The 2-line behavior
change with the right framing.

## What I learned

The "strict provider exposes a bug that lenient providers were silently
hiding" pattern is generic and worth recognizing: when you have one
handler signature consumed by N providers and N-1 of them tolerate
unknown fields, the strict one is doing you a *favor* by surfacing
the bug. The OpenAI 400 here was the diagnostic that the codepath had
been silently sending wrong fields to every other provider for a long
time — they just didn't complain. The fix isn't "make it look more
like the other providers"; it's "actually scrub the params at the
boundary so even the strict providers are happy."
