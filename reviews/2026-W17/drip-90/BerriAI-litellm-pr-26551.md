---
pr: 26551
repo: BerriAI/litellm
sha: ec73b7c7707f689f9dddf67ce9aa2190fddd62a5
verdict: merge-after-nits
date: 2026-04-27
---

# BerriAI/litellm #26551 — fix(guardrails): re-emit chunks in tool_permission streaming hook when no tool_calls found

- **Head SHA**: `ec73b7c7707f689f9dddf67ce9aa2190fddd62a5`
- **Author**: someswar177
- **Size**: small (5-line fix in `tool_permission.py` + a regression test) — bundled with unrelated formatter-only diffs in `factory.py`, `predibase/transformation.py`, `xecguard.py`

## Summary
The streaming-iterator hook had a bare `return` inside an `async generator` when no tool_calls were detected in the assembled response. In Python, `return` inside an async generator terminates the generator without yielding — the upstream LLM chunks were silently swallowed and the client received only `data: [DONE]`. Fix re-wraps the assembled response in a `MockResponseIterator` and yields its chunks before returning.

## Specific findings
- `litellm/proxy/guardrails/guardrail_hooks/tool_permission.py:683-690` — fix is exactly right:
  ```python
  mock_response = MockResponseIterator(model_response=assembled_model_response)
  async for chunk in mock_response:
      yield chunk
  return
  ```
  This converts the *assembled* (non-streamed) `ModelResponse` back into chunks the SSE client expects. Two things to double-check:
  1. Chunk shape parity — `MockResponseIterator` produces a single big chunk (or N small ones?) compared to the original streamed deltas. If clients are accumulating `delta.content` on the assumption of multiple deltas, a single fat chunk is still spec-compliant but may surprise tooling that auto-detects "first non-empty chunk" timing for TTFB metrics.
  2. The `assembled_model_response` is built earlier in the function via `stream_chunk_builder`. Reassembling and *then* re-disassembling is wasteful for the common case (most LLM turns have no tool_calls). Worth a follow-up: capture the original chunks in a buffer during the assembly walk and replay them, instead of `MockResponseIterator(assembled)`.
- `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_tool_permission.py:494-532` — regression test correctly drives the empty-tool-calls path and asserts `len(chunks) >= 1`. Good. Two nits:
  - `assert len(chunks) >= 1` is a weak assertion. It catches the bare-`return` regression but not "we yielded one empty chunk with no content". Strengthen to `assert any(getattr(c, 'choices', None) for c in chunks)` or assert reconstructed text equals `"Hello, world!"`.
  - The `text_chunk` fixture has `choices=[]`. If the implementation ever decides to fast-path on "empty choices → skip", the test would silently pass for the wrong reason. Use a chunk with one realistic choice delta.
- `litellm/litellm_core_utils/prompt_templates/factory.py:5291-5296`, `litellm/llms/predibase/chat/transformation.py:175-350`, `litellm/proxy/guardrails/guardrail_hooks/xecguard/xecguard.py:212-220, 588` — these are all pure black/ruff line-wrap reformats with zero behavior change and no relation to the tool_permission fix. They should be split into a separate `chore: format` PR. As-is, `git blame` for those lines will forever point at "fix(guardrails): re-emit chunks…", which is misleading. Also the `xecguard.py` change is a no-newline-at-EOF fixup (`\ No newline at end of file`) — also unrelated.

## Risk
Low for the actual fix — it's a correctness restoration. The bundled formatter diffs add zero risk but pollute the commit history.

## Verdict
**merge-after-nits** — accept the tool_permission fix as-is, ask the author to (a) strengthen the regression assertion, (b) split the factory/predibase/xecguard formatter diffs into a separate PR.
