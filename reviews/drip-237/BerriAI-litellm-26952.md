# BerriAI/litellm#26952 — Refactor content traversal in moderation/guardrail hooks

- **PR**: https://github.com/BerriAI/litellm/pull/26952
- **Head SHA**: `cb5957ab517bf8e071ab1622a5f8c6ff6d6f9594`
- **Size**: +722 / -101, 10 files (new `safe_messages.py` helper, six guardrail/moderation hooks updated, two test files)
- **Verdict**: **merge-after-nits**

## Context

Chat-completion `message["content"]` may be a plain string OR a list of content parts (multimodal: `[{"type": "text", "text": "..."}, {"type": "image_url", ...}]`). Six guardrail hooks (`banned_keywords`, `google_text_moderation`, `openai_moderation`, `secret_detection`, `azure_content_safety`, `lakera_ai_v2`, `ibm_guardrails`) all carried a copy-paste `if isinstance(content, str)` predicate that *silently skipped* the list shape. Net effect: any client could bypass moderation/secret-detection/banned-keyword checks by sending the request as `content: [{"type": "text", "text": "<bypass payload>"}]` instead of `content: "<bypass payload>"`. This is a real bypass primitive — the test file `test_multimodal_content_bypass.py` (320 lines, all new) exists for a reason.

## What's right

- **Single source of truth for content traversal.** New `litellm/litellm_core_utils/safe_messages.py:1-132` exports `iter_message_texts(messages) → Iterator[(msg_idx, part_idx_or_None, text)]`, `set_message_text(messages, msg_idx, part_idx, new_text)`, `iter_request_texts(data)`, `set_request_text(data, source, slot, new_text)`, and `collect_message_text(messages)`. The slot-tuple `(msg_idx, part_idx)` design correctly handles the "string content" vs "list-of-parts content" duality with one call site per consumer.
- **The traversal handles three real shapes.** `iter_message_texts` yields for: (a) string content, `part_idx=None`; (b) list with `{"type": "text", "text": "..."}` parts; (c) list with bare-string entries (the bedrock shape). Non-string `text` values inside a part dict are skipped silently — correct fail-quiet for malformed input that isn't the hook's responsibility to validate.
- **Defensive `set_message_text` at `safe_messages.py` (lines around 60-90)** — bounds-checks `msg_idx`, type-checks `messages[msg_idx]` is a dict, and is no-op-on-shape-drift so a concurrent mutation between `iter_*` and `set_*` doesn't crash the hook. Right call for a guardrail layer (drop the redaction silently rather than 500 the request).
- **`secret_detection.py` collapse is the headline win.** The prior 71-line copy-pasted `messages` / `prompt` (str) / `prompt` (list) / `input` (str) / `input` (list) chain at `:477-547` is replaced with one 16-line `for source, slot, text in iter_request_texts(data): ... set_request_text(data, source, slot, redacted)` loop at `:480-495`. Same behavior surface, one traversal helper, one redaction-write helper, one source-of-truth for "which keys count as request text."
- **Test coverage matches the threat model.** `tests/test_litellm/proxy/guardrails/test_multimodal_content_bypass.py` (320 lines, new file) and `tests/test_litellm/litellm_core_utils/test_safe_messages.py` (166 lines, new file) — pinning both the helper contract in isolation and the end-to-end "list-of-parts payload triggers each guardrail" property for every refactored hook. This is the correct test strategy for a fail-closed refactor.

## Risks / nits

- **`iter_request_texts` source-string `"messages" | "prompt" | "input"` is documented in the module docstring but not as an enum/Literal type.** A typo at the call site (`set_request_text(data, "message", slot, ...)`) silently no-ops. Worth promoting to `Literal["messages", "prompt", "input"]` so the type-checker catches drift.
- **`iter_message_texts` does not yield `tool_call.function.arguments`, `tool_call.function.name`, or assistant-message `tool_calls[]`.** The PR scope is reasonable (user-controlled text only) but a banned-keyword payload smuggled into a function-call argument string the model will echo back is still a bypass surface. Worth a one-line PR-body note explicitly excluding tool_calls from scope so reviewers don't assume coverage.
- **`secret_detection.py` previously emitted `verbose_proxy_logger.debug("Data after redacting input %s", data)` at the input-list arm.** The new collapsed loop drops that debug log. Cosmetic but worth restoring for parity if anyone has a debug-time trace pattern grepping for the string.
- **`google_text_moderation.py:96-100`** — `text = collect_message_text(messages)` concatenates without separator. Old code did `text += m["content"]` (also no separator), so behavior is byte-identical, but joining adjacent messages without a `\n` boundary is a long-standing semantic concern (Google Text Moderation flags context-dependent phrases differently when message boundaries collapse). Out of scope for this PR but worth flagging.

## Verdict

**merge-after-nits.** Clean centralization of a real bypass primitive (multimodal content shape silently skipping six guardrail hooks). Test coverage matches the threat model. Address the `Literal` type pin and the tool_calls scope note before merge; the `verbose_proxy_logger.debug` parity drop and the message-join-separator concern can be follow-ups.
