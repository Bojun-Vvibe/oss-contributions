# BerriAI/litellm #27085 — feat(proxy): optional Natasha guardrail for Russian person names (PER)

- PR: https://github.com/BerriAI/litellm/pull/27085
- Author: roman02s
- Head SHA: `ea13cbe89907473d5bbc949d603e0830cf5b58ea`
- Updated: 2026-05-03T12:50:34Z

## Summary
Adds an optional `natasha_ru_person` proxy guardrail that uses the on-device Natasha NER tagger to detect and redact Russian person-name (PER) spans in `user`/`system` message content during the `pre_call` hook. Ships behind a `litellm[natasha-ru-person]` extra so the default install stays lean. Includes user docs, the guardrail integration module, and (per docs) a unit test under `tests/test_litellm/proxy/guardrails/test_natasha_ru_person.py`.

## Observations
- `docs/my-website/docs/proxy/guardrails/natasha_ru_person.md` is unusually thoughtful for a guardrail doc: it explicitly calls out (lines ~12-15) that detection runs as **local NLP inference** with no external HTTP call and no runtime weight download. That's the right thing to surface up-front for an enterprise audience evaluating data-egress risk. It also honestly disclaims (lines ~17-19) that it's not a secret-scanner replacement and won't perfectly disambiguate surnames vs toponyms — both true and load-bearing.
- `litellm/proxy/guardrails/guardrail_hooks/natasha_ru_person/__init__.py` line ~22-29: the `mode` normalization handles `list`, `str`, and fallthrough-to-`pre_call` defensively. Good. But `GuardrailEventHooks(mode)` will raise `ValueError` if a user supplies an unknown mode string — wrap it with a clearer error message ("natasha_ru_person only supports `pre_call`; got `<mode>`") since the doc explicitly calls out `pre_call` as the only supported hook.
- Same file lines ~32-37: `redaction_placeholder` is read from two attribute names (`natasha_redaction_placeholder` and `natasha_ru_person_redaction_placeholder`). Picking up two aliases is friendly but undocumented — either document both spellings in the env-vars / config table or drop one to avoid drift.
- The `NATASHA_RU_PERSON_PLACEHOLDER` env var documented in the doc (line ~50) and the two `optional_params` attribute names in `__init__.py` need a clear precedence order spelled out (env vs per-guardrail config). Right now it's ambiguous which wins.
- The `pre_call` event hook is the correct choice — running PII redaction on stream output (`during_call`) would be wrong for a span-based replacement. Confirm in tests that PER spans inside list-of-text-chunks `content` are also handled (the doc claims "string or list of `{ "type": "text", "text": "..." }` chunks") — that's the case most likely to regress.
- Optional-extra packaging (`litellm[natasha-ru-person]`) is the right pattern for a heavyweight NLP dependency; this keeps the default install footprint unchanged.

## Verdict
`merge-after-nits`
