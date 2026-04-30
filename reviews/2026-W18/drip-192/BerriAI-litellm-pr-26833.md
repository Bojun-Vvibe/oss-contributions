# BerriAI/litellm #26833 — Propagate cache_control through Responses API → Chat Completion transform

- **URL:** https://github.com/BerriAI/litellm/pull/26833
- **Head SHA:** `bcedf79b0f3041a78574ec1b67303d24c848eb87`
- **Files:** `litellm/responses/litellm_completion_transformation/transformation.py` (+22/-19), `tests/test_litellm/responses/litellm_completion_transformation/test_litellm_completion_responses.py` (+~94)
- **Verdict:** `merge-after-nits`

## What changed

When a Responses API request flows through litellm's "transform to Chat Completion under the hood" path (used to back the Responses API on providers that don't natively implement it — e.g. cache-aware Anthropic-compatible Chat Completion endpoints), `cache_control` fields on either the input *item* or individual content *blocks* (text / `input_file` / `input_image`) were silently dropped during the transform. Net effect: callers paying for prompt caching via the Responses API on a Chat-Completion-backed provider got *no caching* on this transform path.

The fix is propagation at four points in `_transform_responses_api_input_item_to_chat_completion_message` / `_transform_responses_api_content_to_chat_completion_content`:

1. **Message-level cache_control** at `transformation.py:991-1000`:
   ```python
   message = GenericChatCompletionMessage(role=..., content=...)
   cache_control = input_item.get("cache_control")
   if cache_control is not None:
       message["cache_control"] = cache_control  # type: ignore[typeddict-unknown-key]
   return [message]
   ```
   Replaces the previous one-shot `return [GenericChatCompletionMessage(...)]` at the same location.

2. **`input_file` block-level cache_control** at `:1281-1287`. Same pattern: build the block, conditionally add `cache_control` if present on the input item, append.

3. **`input_image` block-level cache_control** at `:1289-1297`. Same pattern.

4. **Generic text block cache_control** at `:1303-1311`. Same pattern, applied to the `else:` branch that handles non-`input_file` / non-`input_image` content items (text and `input_text`).

Test additions cover all four propagation paths plus a negative ("absent when not provided") assertion in `TestResponsesAPItoChatCompletion`-style class — five new test methods at `test_litellm_completion_responses.py:992-1088`.

## Why it's right

- **Real bug fix with a clear root cause.** The previous code at the deleted shape (`return [GenericChatCompletionMessage(role=..., content=...)]` and `content_list.append(LiteLLMCompletionResponsesConfig._transform_input_file_item_to_file_item(item))`) constructed the output dict from a fixed argument set with no escape for arbitrary passthrough fields. `cache_control` on the *input* item had no path to the *output* item — it was silently elided. This PR doesn't change behavior for callers who weren't sending `cache_control`; it adds an opt-in propagation path for callers who were.
- **Per-item conditional add is the right shape.** Doing `if cache_control is not None: block["cache_control"] = cache_control` per branch (rather than a post-hoc copy at the end) keeps each branch self-contained — easy to delete if a future content type doesn't support caching, easy to understand at the branch site. The alternative (a generic `_propagate_passthrough_fields(item, block)` helper) would be more general but obscures *which* fields are being passed through; given there's currently exactly one such field, per-branch wins on readability.
- **`type: ignore[typeddict-unknown-key]` at `:1000`** for the message-level case is correct — `GenericChatCompletionMessage` is a `TypedDict` that doesn't declare `cache_control`. Adding it to the TypedDict definition would force every callsite to optionally consider it; the local ignore confines the type-system stretch to this one site. The block-level cases (`input_file`, `input_image`) at `:1287` / `:1297` are pure `dict` so don't need the ignore.
- **Tests cover the right cells.** Five tests = (text block has it) × (input_file block has it) × (input_image block has it) × (message-level has it) × (absent passes through cleanly). The mixed-content test (`test_cache_control_propagated_to_content_block` at `:992-1009`) explicitly asserts only the *first* block has `cache_control`, the second doesn't — locking the contract that this is per-item, not per-content-list.
- **`assert "cache_control" not in result[1]`** in three of the five tests is the load-bearing negative assertion. Without it, a future "convenience" patch that defaults `cache_control = {"type": "ephemeral"}` everywhere would not fail any test. The negative covers it.

## Nits

1. **Variable shadowing is reused but the previous pattern wasn't.** All four propagation sites bind `cache_control = item.get("cache_control")`. Inside the `_transform_responses_api_content_to_chat_completion_content` loop at `:1278-1311`, this rebinds `cache_control` once per loop iteration. Fine. But the message-level binding at `:996` and the block-level bindings reuse the same name. A shadow-aware linter probably won't flag it since they're in different scopes; if `mypy --strict` is in CI, no issue. Worth a glance.

2. **No round-trip test through the *full* Chat Completion request shape.** All five new tests assert the *transform* output. None asserts that the resulting `cache_control` field actually survives through the next layer (e.g. Anthropic provider transform, which is the actual consumer that converts `cache_control` to the Anthropic-native `cache_control: {"type": "ephemeral"}` block-marker shape). A single integration-style test that goes Responses-API-input → final-provider-payload and asserts the cache-marker actually reaches the provider request would lock the *value* contract, not just the *propagation* contract.

3. **`cache_control = {"type": "ephemeral"}` is the only value tested.** Anthropic's spec also supports `{"type": "ephemeral", "ttl": "1h"}` with a TTL field. The propagation code uses `cache_control` as opaque-passthrough so it'll work for any shape — but a single test asserting opaque passthrough of an unknown shape (e.g. `{"type": "ephemeral", "ttl": "5m", "future_field": "x"}`) would prevent a future "validate cache_control shape before passthrough" change from accidentally narrowing the contract.

4. **`block` variable name collision.** The `else:` branch at `:1303` rebinds `block` to a `dict` (with `type` / `text` keys), but the immediately-preceding `input_image` branch at `:1289` also rebound `block` to `dict(...)`. Re-using the name across mutually-exclusive branches is fine, just slightly noisy. If a future patch accidentally drops the `else:` (turning it into a fallthrough), the rebind history would matter — minor refactor risk.

5. **`message["cache_control"] = cache_control  # type: ignore[typeddict-unknown-key]`** at `:1000`. Long-term, declaring `cache_control: NotRequired[Mapping[str, Any]]` on `GenericChatCompletionMessage` would be cleaner than a per-callsite ignore. That's a separate-PR refactor — this PR is correctly scoped.

## Risk

Low. Pure additive propagation — callers without `cache_control` on the input get exactly the previous output (verified by `test_cache_control_absent_when_not_provided` at `:1080-1093`). Callers *with* `cache_control` start getting it propagated, which was their stated intent. Worst case if the downstream provider transform doesn't recognize `cache_control` on the message/block, it silently drops there too — same as today, no regression. The fix unblocks prompt-cache cost savings for Responses-API-on-Chat-Completion-backed-providers callers, which is a real money-on-the-table item.
