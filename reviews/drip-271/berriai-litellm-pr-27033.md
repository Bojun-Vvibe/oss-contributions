# BerriAI/litellm #27033 — Litellm ishaan april7 2

- URL: https://github.com/BerriAI/litellm/pull/27033
- Head SHA: `a9014650d6fa622851835d2594eee67ab40bfab0`
- Author: @ishaan-berri
- Stats: large grab-bag — adds `VoyageMultimodalEmbeddingConfig`
  (~225 new LOC), threads `prompt` param + non-string serialization
  through ElevenLabs Scribe transcription, edits `__init__.py` and
  `_lazy_imports_registry.py`

## Specific feedback

- **PR title is uninformative** ("Litellm ishaan april7 2"). For an
  actually-merge-blocking grab-bag PR, retitle to enumerate the changes
  (`feat(voyage): add multimodal embedding config; feat(elevenlabs):
  forward prompt param + serialize non-string form values`). Reviewers
  cannot triage from the current title.
- `litellm/__init__.py:1605-1610` and `litellm/_lazy_imports_registry.py:223,
  886-890` — `VoyageMultimodalEmbeddingConfig` registered both eagerly
  (in the conditional import block) AND lazily. That's the existing
  pattern for the sibling Voyage configs, so it's consistent — but
  worth confirming there's no double-init footgun (the lazy-import
  table is the source of truth at runtime per litellm's import design).
- `litellm/llms/voyage/embedding/transformation_multimodal.py:1-225`
  (new file) — the URL-routing logic at `get_complete_url:48-58` will
  *append* `/multimodalembeddings` to a custom `api_base` only if the
  base doesn't already end with that segment. Edge case: if a user
  passes `api_base="https://proxy.internal/voyage/multimodalembeddings/"`
  (trailing slash), the suffix check fails and you append a duplicate
  segment. Strip trailing `/` first.
- `transformation_multimodal.py` — `map_openai_params:69-78` maps
  `encoding_format → output_encoding` and `dimensions → output_dimension`.
  Verify Voyage's actual API spec: `encoding_format=base64` (a
  legitimate OpenAI value) — does Voyage's `output_encoding` accept
  `base64`, or only `float`/`raw`? If not, drop_params should be
  honoured here.
- `litellm/llms/elevenlabs/audio_transcription/transformation.py:35`
  — adds `prompt` to supported params. Comment "ElevenLabs Scribe
  accepts it as vocabulary context" is the right kind of justification
  — please cite the ElevenLabs API doc URL inline so a future
  maintainer can verify.
- `transformation.py:65-77` — new `_serialize_form_value` correctly
  handles bool (lowercase), list/dict (JSON), and falls back to `str()`.
  Bool→lowercase is the standard multipart convention. Good helper.
- `transformation.py:106-107, 118` — both call-sites swapped from
  `str(value)` to `_serialize_form_value(value)`. Symmetric application,
  no regression for plain string values (str→str is identity).
- **Missing tests entirely.** New 225-line provider config and a
  serialization helper are zero-tested in this diff. At minimum: one
  unit test for `_serialize_form_value` (bool/list/dict/string/int) and
  one for `VoyageMultimodalEmbeddingConfig.get_complete_url` covering
  the trailing-slash edge case above.
- Bundling unrelated changes in one PR (Voyage multimodal + ElevenLabs
  prompt + ElevenLabs serialization) makes review and bisect harder.
  Suggest splitting.

## Verdict

`request-changes` — substantive content but multiple correctness gaps
(api_base trailing-slash, encoding_format mapping, missing tests) plus
an opaque title and bundled scope. Split into two PRs, add tests, and
the per-feature pieces are individually mergeable.
