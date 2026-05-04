# BerriAI/litellm PR #27101 — fix(core_helpers): map vLLM 'repetition' finish_reason to 'stop'

- **Link:** https://github.com/BerriAI/litellm/pull/27101
- **Head SHA:** `9a18172d371f50603f69ed58ac636fd7259354f3`
- **Verdict:** `merge-as-is`

## What it does

Single-line addition to the `MAP_FINISH_REASON_TO_OPENAI_FINISH_REASON` table
at `litellm/litellm_core_utils/core_helpers.py:98-99`:

```python
# vLLM (server-side RepetitionDetectionParams cuts off contiguous loops)
"repetition": "stop",
```

Plus a one-line test class at
`tests/test_litellm/litellm_core_utils/test_core_helpers.py:146-152`
asserting `map_finish_reason("repetition") == "stop"`.

## Design analysis

- **Right destination value.** vLLM's `RepetitionDetectionParams` is
  server-side n-gram repetition detection that *cuts off* generation when
  the model loops on contiguous tokens. Semantically the model decided to
  keep going but the server forced an end — that's closer to OpenAI's
  `stop` (natural end / stop-token-hit) than `length` (max_tokens hit) or
  `content_filter` (safety filter). Either `stop` or `length` would be
  defensible; `stop` is the better default because downstream consumers
  treat `length` as "incomplete, consider continuing" which would prompt
  retry loops on a server that *just decided* further generation was
  pathological.
- **Surgical placement in the table.** Drops in next to the bedrock
  `guardrail_intervened` and the OpenAI passthrough block — the right
  semantic neighborhood (provider-specific terminator → canonical OpenAI
  string).
- **Side benefit:** silences the `"Unmapped finish_reason 'repetition'"`
  warning on every affected response row that the test docstring (line
  149-152) calls out.

## Risks / nits

Effectively none. Three observations for completeness:

1. The test class only covers the happy path. A negative-property test
   asserting `"repetition" not in {"length", "content_filter"}` after
   mapping would lock the design choice against drift if someone later
   "corrects" it to `length`.
2. vLLM also can emit `stop_str` and (in some versions) `eos_token` —
   those are already mapped via the `stop` → `stop` passthrough at line 99
   and the implicit fallthrough, but it's worth a follow-up doc PR
   listing all vLLM-specific finish reasons in one place.
3. No release-notes touch in this diff — assumed handled out of band.

## What I learned

This is the canonical "map-table addition is the smallest possible bug
fix" PR. The table-driven design at `core_helpers.py:91-105` makes
provider-specific extensions a one-line change with one-line test, which
is exactly how this layer should evolve as new backends ship new
finish-reason vocabulary. The alternative (provider-specific override
methods) would have made this a 50-line change touching multiple files.
