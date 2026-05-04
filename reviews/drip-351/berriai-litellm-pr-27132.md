# BerriAI/litellm PR #27132

- **Title**: refactor(anthropic,bedrock): hoist drop_params output_config warning to a single constant
- **Author**: mateo-berri
- **Head SHA**: `98f6e5e72c94e668f7da343b6385028976ea67c7`
- **Diff**: +14 / -10 (3 files)

## Summary

Pure refactor: extracts the duplicated "Dropping unsupported `output_config`" warning string into `DROP_UNSUPPORTED_OUTPUT_CONFIG_WARNING` in `anthropic/chat/transformation.py` and re-imports it from the two bedrock sites.

## File-by-file

### `litellm/llms/anthropic/chat/transformation.py` (L9-13, L22-25)

Constant defined at module top-level (L9-13) with the `%s` format specifier preserved so existing `verbose_logger.warning(template, model)` call sites stay correct. Substitution at L22-25 is mechanical and matches the original wording verbatim.

### `litellm/llms/bedrock/chat/converse_transformation.py` (L37, L44-49)

Adds `DROP_UNSUPPORTED_OUTPUT_CONFIG_WARNING` to the existing import block — clean. Note: original bedrock string at L45-47 was line-broken slightly differently from the anthropic one ("Effort is only supported on" vs "Effort is only supported on Opus 4.5+,"). The hoisted constant uses the anthropic wording. Net log content is identical, just whitespace is normalized — non-issue.

### `litellm/llms/bedrock/messages/invoke_transformations/anthropic_claude3_transformation.py` (L61-64, L71-75)

Same pattern. Import added cleanly to the existing `from ... anthropic.chat.transformation import (...)` block.

## Observations

- All three call sites pass `model` as the lone format arg, matching the single `%s`. Verified by inspection.
- No behavior change. No tests needed/added (refactor only).
- Constant is marked module-level, not class-level — appropriate since the bedrock paths import without instantiating `AnthropicConfig`.

## Nits

- Could consider colocating with `REASONING_EFFORT_TO_OUTPUT_CONFIG_EFFORT` (already imported alongside in converse_transformation) since they're both effort-related constants. Minor.

## Verdict

**merge-as-is**
