---
pr: 26505
repo: BerriAI/litellm
sha: 37bb6c11567c0a271205233f2485189740aa2915
verdict: merge-after-nits
date: 2026-04-26
---

# BerriAI/litellm#26505 — feat(fireworks): extract <think> tags into reasoning_content

- **URL**: https://github.com/BerriAI/litellm/pull/26505
- **Author**: anuragrao04

## Summary

Adds `<think>...</think>` extraction to the Fireworks AI provider so
reasoning models (e.g. `accounts/fireworks/models/deepseek-r1`,
`qwq-32b`) surface reasoning text in `message.reasoning_content`
instead of leaking it into `message.content`. Mirrors existing
DeepSeek/Ollama behavior. Two surfaces:

1. Non-streaming: post-process `transform_response` to call
   `_extract_reasoning_content` on every choice.
2. Streaming: new `FireworksAIChatCompletionStreamingHandler`
   subclass of `OpenAIChatCompletionStreamingHandler` with a
   `chunk_parser` that tracks `started_reasoning_content` /
   `finished_reasoning_content` across deltas.

## Reviewable points

- `litellm/llms/fireworks_ai/chat/transformation.py:406-414` — the
  non-streaming path checks
  `getattr(_msg, "reasoning_content", None) is None` before
  overwriting, so providers that already populate
  `reasoning_content` natively (current Fireworks behavior for
  some models) are not clobbered. Right defensive check.

- `transformation.py:423-432` — `get_model_response_iterator`
  override returns the new handler. Note the parent class signature
  must match: confirmed the base method in
  `litellm/llms/openai/chat/gpt_transformation.py` takes the same
  `(streaming_response, sync_stream, json_mode)` triple. Compatible.

- `transformation.py:507-540` — the streaming state machine has a
  subtle bug at the `</think>` branch:
  ```py
  if "</think>" in content and self.started_reasoning_content:
      parts = content.split("</think>", 1)
      reasoning_chunk = parts[0]
      content_after = parts[1] if len(parts) > 1 else ""
      ...
      delta["reasoning_content"] = reasoning_chunk
      delta["content"] = content_after if content_after else None
  ```
  The split discards the `</think>` delimiter, which is correct.
  But `started_reasoning_content` is *not* reset to `False` after
  the close tag. If a model emits a second `<think>...</think>`
  block in the same stream (some R1-style models do this for
  tool-call planning), the next `<think>` opener will hit
  `started_reasoning_content == True` and the
  `if "<think>" in content` branch will strip the tag but the
  `elif self.started_reasoning_content and not self.finished`
  guard then routes the chunk to `reasoning_content` — except
  `finished_reasoning_content` is also still `True` from the
  previous block, so the chunk falls to the `else: delta["content"]
  = content` arm. Net: the second reasoning block leaks into
  `content`. Reset both flags on close, or treat them as a
  single "in_thinking" boolean.

- Same path: a `</think>` that arrives *without* any prior
  `<think>` (model output starts with a closing tag — yes this
  happens with quantized R1 derivatives) will hit the `if
  "</think>" in content and self.started_reasoning_content`
  guard, evaluate to `False`, and fall through to the `else`
  arm. So content like `"</think>actual answer"` is preserved
  intact. That's the correct conservative behavior. Worth a unit
  test pinning it.

- The extraction does no validation that the buffer between
  `<think>` and `</think>` actually fits in one chunk. With
  small chunk sizes (typical for Fireworks: 1-3 tokens per
  chunk), the streaming handler will run through many
  mid-stream chunks before hitting `</think>` — the
  `elif self.started_reasoning_content and not self.finished`
  arm handles that, routing each mid-stream chunk to
  `reasoning_content`. Confirmed.

- `chunk_parser` returns `ModelResponseStream(**kwargs)` and
  swallows nothing — exceptions bubble. The bare `except
  Exception as e: raise e` at the bottom is dead code and should
  be removed.

## Rationale

Right shape, mirrors the DeepSeek/Ollama precedent, but the
multi-`<think>`-block state-reset bug is a real correctness
issue for R1-class models that plan-then-act. Fix the reset and
add a regression test for `<think>...</think><think>...</think>`
streaming, then ship.

## What I learned

Reasoning-tag extraction is the kind of feature where the test
suite needs to cover at least four cases: (a) clean
`<think>x</think>y`, (b) split across chunks, (c) multiple
blocks in one stream, (d) close-without-open. Most
implementations only test (a) and (b), miss (c), and silently
break R1-style multi-step reasoning.
