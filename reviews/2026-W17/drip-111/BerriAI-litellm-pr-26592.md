# BerriAI/litellm PR #26592 — filter empty text blocks from assistant messages on Bedrock invoke

- **PR**: https://github.com/BerriAI/litellm/pull/26592
- **Author**: @weiguangli-io
- **Head SHA**: `d958282f297ea458785d58b85869ff628eec72af`
- **Size**: +20 / −0
- **Files**: `litellm/llms/bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py`

## Summary

Adds a step "5b" to `transform_anthropic_messages_request` that walks `messages` and drops `{"type": "text", "text": ""}` (or whitespace-only) blocks from any assistant message whose `content` is a list. Bedrock's invoke endpoint rejects these with a 400 ("The text field in the ContentBlock object is blank."). The empty blocks come from streamed responses where `content_block_start` emits an empty `text` placeholder before the first delta.

## Verdict: `merge-after-nits`

The fix is correct and the linked issue (#26554) confirms the upstream Bedrock 400. Two things keep it from `merge-as-is`: there's no regression test in this PR, and the filter operates only on assistant turns even though the same shape can theoretically appear on a `user` turn whose content was assembled programmatically from a tool-result chain. The asymmetry is probably fine in practice but should be a deliberate decision, not an accident.

## Specific references

- `litellm/llms/bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py:516-533` — the filter is a list-comprehension over `msg["content"]` keeping every block *unless* it's a dict with `type == "text"` AND `text` strips to empty. The `not block.get("text", "").strip()` is the right whitespace-tolerant predicate (matches `""`, `" "`, `"\n"` — all rejected by Bedrock for the same reason). Tool-use, tool-result, image, and reasoning blocks pass through untouched because they don't have a `type == "text"` field.
- `:518-519` — the role gate is hard-coded to `"assistant"`. The upstream Bedrock validator runs against *every* message in the array, so a user message assembled by a downstream caller from a chain of `[text, tool_result, text]` parts that includes an empty text block would still 400. Most callers don't generate that shape, but the LangChain / Vercel-AI bridges sometimes produce empty-text `user` parts when concatenating mixed-content turns. Worth either widening the role check to `("assistant", "user")` or documenting why assistant-only is the chosen scope.
- `:521` — `isinstance(msg.get("content"), list)` correctly skips string-content shorthand (`{"role": "assistant", "content": "hi"}`). Good — that path can't produce an empty *block* by construction.
- `:516` — the comment block correctly identifies the bug as a streaming-side artifact ("Claude's streamed responses that emit a leading content_block_start with empty text") and links #26554. That's the right level of context for a maintainer skimming the diff a year later.
- No test file is touched. A 5-line pytest case that constructs a request with `[{"type":"text","text":""},{"type":"tool_use",...}]` and asserts the empty block is dropped before transform-out would close the regression door cheaply. The test scaffolding for `anthropic_claude3_transformation.py` already exists (`tests/llms/bedrock/test_anthropic_invoke_transformation.py` is the convention); adding one parametrized case is mechanical.

## Nits

1. Add a regression test (single parametrized case is enough).
2. Either widen the role gate to also cover `"user"`, or comment why assistant-only is sufficient.
3. The list-comprehension would read more cleanly as a small helper `_drop_empty_text_blocks(content)` so the next "filter empty foo" change has a hook to extend.

## What I learned

This is a textbook "the upstream provider's stricter validator catches a streaming-side artifact that the source provider tolerates" bug — Anthropic's own API silently accepts the empty-text block; Bedrock's invoke surface 400s. The cure is always to scrub the canonical shape *before* the provider-specific transformer hands the payload off. Doing the scrub inside the Bedrock-specific transformer (rather than in the canonical request layer) is the right scope: only Bedrock cares, and adding a generic scrub would hide the streaming-side bug from anyone debugging a non-Bedrock provider that legitimately tolerates it.
