# PR #26621 — fix(health-check): skip max_tokens for non-chat modes

- **Repo**: BerriAI/litellm
- **PR**: #26621
- **Head SHA**: `500b2a4903998eae661226e2825a1dd281bcc728`
- **Author**: udit-rawat (Udit)
- **Size**: +55 / -3 across 2 files (fixes #26406)
- **Verdict**: **merge-as-is**

## Summary

Closes the false-unhealthy ghost in the proxy UI for image,
embedding, rerank, audio, OCR, search, and moderation
deployments. The health check was unconditionally injecting
`max_tokens` via `_resolve_health_check_max_tokens` for every
deployment, but those non-chat endpoints reject `max_tokens` with
a 400, so the UI marked working models as permanently unhealthy.
Fix is a mode-set membership check that skips the injection
entirely when `model_info.mode` is one of the 10 non-chat modes.

## Specific changes

- `litellm/proxy/health_check.py:35-46` — defines a module-level
  `_NON_CHAT_MODES` frozenset-style `set` covering the 10 modes
  that reject `max_tokens`: `image_generation`, `image_edit`,
  `video_generation`, `embedding`, `rerank`, `audio_transcription`,
  `audio_speech`, `ocr`, `search`, `moderation`. Naming and
  scope are right — module-private, set for O(1) lookup, no
  hidden state.
- `litellm/proxy/health_check.py:380-388` — inside
  `_update_litellm_params_for_health_check`, reads `_mode =
  model_info.get("mode", None)` and gates the existing
  `_resolve_health_check_max_tokens` + assignment block behind
  `if _mode not in _NON_CHAT_MODES`. The unknown/None mode case
  falls through to the *include* path, which preserves
  backwards-compatibility for callers that don't declare a mode
  (treated as chat by default — the right default).
- `tests/test_litellm/proxy/test_health_check_max_tokens.py:88-122`
  adds a parametrized test asserting `max_tokens` is *absent*
  from `updated_params` for each of the 10 modes. Uses
  `monkeypatch.setattr(hc_module, "BACKGROUND_HEALTH_CHECK_MAX_TOKENS",
  None)` and `..._REASONING, None` to neutralize env-var-driven
  defaults so the test exercises *only* the new gating branch.
  Good test hygiene — the assertion is a single
  `"max_tokens" not in updated_params`, which fails loudly with
  the offending mode in the message.

## Risks

1. **Drift between `_NON_CHAT_MODES` and the canonical mode
   enum.** `litellm` already has `LlmProviders` / `CallTypes` /
   `LiteLLMRouteType` enums elsewhere; the new `_NON_CHAT_MODES`
   set is a free-standing sibling that future-mode additions
   could miss. Worth a follow-up: build the set from a
   `is_non_chat_mode(mode: str) -> bool` predicate keyed off the
   canonical enum so adding a new mode anywhere updates this
   automatically. Not a blocker — the set is small and the
   coverage is parametrized, so a missed mode is loud-failing.
2. **`mode` is not normalized.** A caller passing
   `"Image_Generation"` or `"image-generation"` (hyphen) silently
   misses the gate and re-introduces the bug. The current
   `model_info` schema docs use the snake_case spellings, so
   this is "garbage in, garbage out", but a `_mode_lower =
   _mode.lower() if _mode else None` + comparison against a
   lowercase set would harden it for free.
3. **`audio_transcription` vs `transcription`.** The PR body's
   bullet list says `transcription` but the implementation uses
   `audio_transcription`. The implementation is correct against
   the canonical mode strings used elsewhere in `health_check.py`,
   but the PR description should be reconciled to avoid future
   confusion.
4. **One Pre-Submission box unchecked.** "Greptile review @ ≥4/5"
   is unchecked. Project-policy decision, not a code-review
   blocker.

## Verdict

`merge-as-is` — minimal correct fix at the exact failure point,
backwards-compatible default (unknown mode → include `max_tokens`
as before), parametrized test pinning all 10 modes. The drift /
normalization nits are follow-up hardening, not blockers.

## What I learned

The most expensive bugs in proxy/healthcheck code are the ones
that turn a correct backend into a *visibly broken* one in the
admin UI — not because the backend stopped working, but because
the probe was malformed. The fix shape ("filter the probe
parameters by mode before sending") is a recurring pattern: any
liveness check that builds a synthetic request body needs to
respect the same conditional-parameter rules the real request
path does. A good follow-up is a single `_build_health_probe_params(mode,
model_info)` helper that owns *all* mode-conditional parameter
shaping in one place, so the next "X mode also doesn't accept Y"
ticket is a one-line change.
