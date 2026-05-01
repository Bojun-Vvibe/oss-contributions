# BerriAI/litellm#26957 — chore(guardrails): cover multimodal + Responses-API content shapes

- **PR**: https://github.com/BerriAI/litellm/pull/26957
- **Author**: stuxf
- **Head SHA**: `41eeaaf6309ba7e983d35705ab04e076085efccc`
- **Files (top)**: `litellm/proxy/guardrails/_content_utils.py` (new), `enterprise/enterprise_hooks/banned_keywords.py`, `enterprise/enterprise_hooks/google_text_moderation.py`, `enterprise/enterprise_hooks/openai_moderation.py`, `enterprise/litellm_enterprise/enterprise_callbacks/secret_detection.py`
- **Verdict**: **merge-after-nits**

## Context

Sister PR to #26952 (drip-237 review). #26952 introduced `litellm/litellm_core_utils/safe_messages.py` with `iter_message_texts` / `set_message_text` / `iter_request_texts` / `set_request_text`. This PR introduces a *separate* helper module `litellm/proxy/guardrails/_content_utils.py` exposing `iter_message_text` (singular) + `walk_user_text`, used by four enterprise hooks (`banned_keywords`, `google_text_moderation`, `openai_moderation`, `secret_detection`). Same bypass primitive: clients sending `content: [{type: "text", text: payload}]` (multimodal list shape) instead of `content: payload` (string shape) silently skip every guardrail that only checks `isinstance(m["content"], str)`.

## What's right

- **`_iter_text_parts_in_content` at `_content_utils.py:14-32`** correctly handles the three real content shapes: bare string, list of `{type: "text", text}` dicts, and list of bare strings (Responses-API mixed-list shape). Empty-string skip via `if content:` / `if part:` / `if text:` is the right discipline — guardrails that fire on empty input are noise.
- **`_coerce_input_to_messages` at `:35-49`** unifies the Responses-API `data["input"]` (which can be `str` | `list[str|dict]` | `list[message-dict]`) into chat-style messages. The branch at `:40-43` correctly distinguishes "list of message dicts (each has `role`)" from "list of content parts" — the role-presence test is the right discriminator since Responses-API messages carry `role` and content parts don't.
- **`iter_message_text` walks every role**, not just `user`. The comment at `:60-62` makes this explicit: `"Walks every role (user, assistant, system, …) — guardrails inspect the entire conversation, not just user turns."` Correct — secret detection in particular needs to scan assistant messages too (model echoing back a leaked secret).
- **`walk_user_text` mutates in-place via index-back rewrite.** `:96-117` rebuilds list-content via a new `new_parts` list and reassigns to `message["content"]`. Crucially, `:104-107` uses `{**part, "text": visit(part["text"])}` rather than `part["text"] = visit(part["text"])` — preserves all other keys (e.g. `type`, `cache_control`, provider-specific fields) which is the right discipline for a content-rewriter that doesn't own the schema. The `secret_detection.py:521-529` index-back-into-`data["prompt"]` fix at `:524-528` (`for idx, item in enumerate(...): ... data["prompt"][idx] = item`) closes a real bug where the prior `for item in data["prompt"]: ... item = item.replace(...)` only rebound the loop variable and left the original list carrying the unredacted secret.
- **Hook-side collapses are uniform.** Each of the four hooks reduces to `for text in iter_message_text(data): self.test_violation(text)` (banned_keywords) or `text = "".join(iter_message_text(data))` (moderation hooks) or `walk_user_text(data, _redact_message_text)` (secret detection). One predicate per hook, no per-shape branching duplication.
- **`secret_detection.py` removes the duplicated `data["input"]` branch** (`:521-543` → comment at `:542-543` `"data["input"] is already covered by walk_user_text above"`). Pre-fix the file had separate `data["messages"]`, `data["prompt"]`, *and* `data["input"]` walks, each with their own list-index bug. Post-fix only the `data["prompt"]` branch remains separate (because the file uses `_redact_message_text` for messages/input via `walk_user_text` but a different per-call inline scan for `prompt`).

## Risks / nits

- **Two parallel helper modules with overlapping responsibility.** `litellm/litellm_core_utils/safe_messages.py` (from #26952) and `litellm/proxy/guardrails/_content_utils.py` (this PR) both walk message text. The naming overlap (`iter_message_texts` vs `iter_message_text`) is a future-confusion landmine — reviewers will land changes in one and not the other. Recommend a follow-up consolidating: one canonical helper module, both call sites import from it. At minimum, a comment at the top of each pointing to the other.
- **`walk_user_text`'s `data["prompt"]` handling is partial.** The function walks `messages` and `input` but the secret-detection file still keeps a separate `data["prompt"]` walk at `:497-519`. Either `walk_user_text` should also cover `prompt` (and the per-hook duplication goes away entirely) or the omission should be documented in the docstring.
- **No coverage for `tool_call.function.arguments` smuggling.** A user-controlled string passed as a tool-call argument is also conversation content the model will eventually see, and a sufficiently determined attacker could smuggle a banned keyword inside `tool_calls[0].function.arguments`. This is the same out-of-scope note that applies to #26952 — worth pinning in the PR body so reviewers don't assume coverage.
- **`_iter_text_parts_in_content` skips non-text parts silently.** For an audio/image guardrail (future), the helper would need extending. A `# TODO: extend for non-text content types` comment at `:32` would signal the intentional scope.

## Verdict

**merge-after-nits.** Closes a real bypass primitive across four enterprise hooks plus a real loop-variable bug in `secret_detection.py`'s `data["prompt"]` walk. Strongest nit: the parallel-helper-module situation with #26952 needs consolidation before they drift further. The `walk_user_text` rewrite-in-place discipline (preserving non-text part keys via `{**part, "text": ...}`) is exactly right.
