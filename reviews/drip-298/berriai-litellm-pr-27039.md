# Review: BerriAI/litellm PR #27039 — fix(anthropic,bedrock): omit thinking/output_config when reasoning_effort="none"

- **Head SHA:** `7f3d7616b7a7d2deda6d6ff8e8f9675d7b50d129`
- **State:** MERGED (2026-05-02)
- **Size:** +74 / -23, 4 files
- **Verdict:** `merge-as-is`

## Summary
Fixes `litellm.APIConnectionError: 'NoneType' object has no attribute 'get'`
that fired whenever a caller set `reasoning_effort="none"` on any
Anthropic-backed chat model. Root cause: `AnthropicConfig._map_reasoning_effort`
returns `None` for `"none"`, but two callers were assigning that `None`
straight into `optional_params["thinking"]`. Downstream
`is_thinking_enabled` then ran `optional_params["thinking"].get("type")`
and threw.

Affected providers: Anthropic direct, Bedrock Converse, Bedrock Invoke
(via `AmazonAnthropicClaudeConfig`), Vertex AI Anthropic
(via `VertexAIAnthropicConfig`), Azure AI Anthropic
(via `AzureAnthropicConfig`).

## What the diff does

### `litellm/llms/anthropic/chat/transformation.py:1088-1116`

Before:
```python
optional_params["thinking"] = AnthropicConfig._map_reasoning_effort(...)
if AnthropicConfig._is_claude_4_6_model(model) or _is_claude_4_7_model(model):
    optional_params["output_config"] = {"effort": mapped_effort}
```

After:
```python
mapped_thinking = AnthropicConfig._map_reasoning_effort(...)
if mapped_thinking is None:
    optional_params.pop("thinking", None)
    optional_params.pop("output_config", None)
else:
    optional_params["thinking"] = mapped_thinking
    if _is_claude_4_6_model(model) or _is_claude_4_7_model(model):
        optional_params["output_config"] = {"effort": mapped_effort}
```

Correct. The "pop both" branch matters specifically for Claude 4.6/4.7 where
`output_config` was being set even though `thinking` was being unset — would
have left a half-configured request.

### `litellm/llms/bedrock/chat/converse_transformation.py:449-458`

`AmazonConverseConfig._handle_reasoning_effort_parameter` calls
`AnthropicConfig._map_reasoning_effort` directly (bypassing
`map_openai_params`), so it needs the same pop-when-None pattern. The patch
applies it cleanly. Note: Bedrock Converse doesn't set `output_config`, so
no pop for that key here — consistent with the rest of the file.

## Test coverage

`tests/test_litellm/llms/anthropic/chat/test_anthropic_chat_transformation.py:2314-2336`
adds `test_reasoning_effort_none_omits_thinking_and_output_config`,
parametrized over four model IDs (`claude-opus-4-5-20251101`,
`claude-opus-4-6-20250514`, `claude-sonnet-4-6-20260219`, `claude-opus-4-7`).
Asserts both keys absent from the result. Good — this is the exact regression
guard the bug needs.

`tests/test_litellm/llms/bedrock/chat/test_converse_transformation.py:288-310`
adds `test_reasoning_effort_none_omits_thinking_for_anthropic_converse`,
parametrized over three Bedrock Converse model IDs.

## What I'd nit if it weren't already merged

1. **Inheritance coverage assertion.** The PR body claims this fix covers
   Vertex AI Anthropic and Azure AI Anthropic via `super().map_openai_params`.
   Worth a one-line parametrized test that constructs `VertexAIAnthropicConfig`
   and `AzureAnthropicConfig` directly and asserts the same omission. Cheap
   insurance against a future override breaking the inheritance chain.

2. **`mapped_thinking is None`** vs. **falsy-check.** The current code is
   strictly None-aware (good), but `_map_reasoning_effort` could in theory
   return `{}` for some future "off" value. Worth a docstring on
   `_map_reasoning_effort` clarifying the contract: `None` means
   "do not enable thinking," anything else is a thinking config.

3. The two unrelated whitespace tidies (`test_max_effort_rejected_for_*`
   one-line `pytest.raises` calls) are fine but should ideally land in their
   own commit. Minor hygiene.

## Verdict rationale
Already merged, and correctly so. The fix targets the exact crash, covers
the cross-provider blast radius via parametrized tests across 7 model IDs,
and the diff stays surgical. The pop-on-None pattern is the right shape to
keep `output_config` and `thinking` from drifting out of sync on the
Claude 4.6/4.7 path. Approve.
