# BerriAI/litellm #26710 — fix(bedrock, anthropic): translate OpenAI file content on tool-result path

- **URL:** https://github.com/BerriAI/litellm/pull/26710
- **Head SHA:** `12e1d02d4e6649da7386a9895ca1ecdc7131f973`
- **Merge SHA:** `d8d1444da42a6abc50074aa890f477943ac37a8b`
- **Files:** `litellm/litellm_core_utils/prompt_templates/factory.py` (+~80 lines: new `_ANTHROPIC_DOCUMENT_BASE64_MEDIA_TYPES` set + `_is_anthropic_document_data_uri` helper at `:1664-1677`, `convert_to_anthropic_tool_result` accepts `AnthropicMessagesDocumentParam` in its return list type at `:1700-1718`, image_url branch at `:1747-1782` now sniffs the data-URI mime type and routes PDF/text data URIs to the document-block path via a synthesized `ChatCompletionFileObject`; `_convert_to_bedrock_tool_call_result` parallel change at `:3977-4042+`), `litellm/types/llms/anthropic.py` (extends `AnthropicMessagesToolResultParam` content union at `:330-340` to include the document param), `tests/test_litellm/litellm_core_utils/prompt_templates/test_litellm_core_utils_prompt_templates_factory.py` (+186 lines of regression cases for tool-result file translation), `tests/test_litellm/llms/bedrock/chat/test_converse_transformation.py` (+221 lines of bedrock-side parallel tests)
- **Verdict:** `merge-after-nits`

## What changed

Fixes a real translation bug on the **tool-result message path**: when a caller sends OpenAI-shaped tool-result content with an `image_url` whose URL is a `data:application/pdf;base64,...` URI (or `data:text/plain;...`), the previous code wrapped it in an Anthropic `type: "image"` block and the upstream API rejected the request because Anthropic's `image` block only accepts image mime types.

The fix at `factory.py:1747-1782`:

1. Extracts the URL string from the `image_url` content (`image_url_value.get("url") if isinstance(image_url_value, dict) else image_url_value` at `:1755-1759`).
2. Calls the new `_is_anthropic_document_data_uri(url_str)` helper at `:1761-1762` to sniff `data:([^;,]+)` against the constant set `{"application/pdf", "text/plain"}` at `:1664`.
3. On hit, synthesizes a `ChatCompletionFileObject` (`synth_file_message` at `:1764+`) and routes it through the document-block translation path that was already used for `type: "file"` content — so the tool-result document path reuses the same translation logic that already worked for the non-tool-result file path, instead of reimplementing it inline.
4. On miss, falls through to the existing image-block translation unchanged — so any image data URI keeps the previous shape, no behavioral drift for the happy path.

The mirror change in `_convert_to_bedrock_tool_call_result` at `:3977-4042` applies the same sniff-and-route logic on the Bedrock Converse path; the type-system extension at `anthropic.py:330-340` widens `AnthropicMessagesToolResultParam.content` to include `AnthropicMessagesDocumentParam` so the new return shape type-checks.

The 186 + 221 = ~407 lines of new tests cover both the Anthropic and Bedrock paths with parameterized cases for `application/pdf`, `text/plain`, image mime types (verifies fall-through), and malformed/missing data URIs.

## Why it's right

- **Sniffs at the right boundary.** The data-URI mime is the *only* place we can distinguish "PDF the caller sent as base64" from "PNG the caller sent as base64" without trusting the caller's `type:` field. The previous code trusted the OpenAI `type: "image_url"` envelope and produced wire-invalid output; the fix sniffs the actual content and routes it to the correct Anthropic block type. That's the right shape for a translation layer that has to bridge two different content-type schemas.
- **The constant set `_ANTHROPIC_DOCUMENT_BASE64_MEDIA_TYPES = {"application/pdf", "text/plain"}` at `:1664` is the right defensive shape.** Anthropic's `select_anthropic_content_block_type_for_file` (referenced in the comment at `:1666-1668`) is the canonical truth for "what mime types does the document block accept" — pinning to that set means *this* code path produces only document-block payloads the API actually accepts. A future Anthropic addition (e.g. `text/csv`) would require updating both this constant and `select_anthropic_content_block_type_for_file` together, which is exactly the right co-located maintenance burden.
- **Synthesizing a `ChatCompletionFileObject` and routing through the existing document-translation code at `:1764+` is the right factoring.** The alternative — inlining the document-block construction at the tool-result site — would create a second, divergent code path for PDF translation. Reusing the existing path means any future fix to the document block (cache_control behavior, etc.) automatically applies to both the standalone-message and tool-result-content cases.
- **Cache-control composition is preserved.** The `add_cache_control_to_content` call in the surrounding image branch (visible in the diff context) is the established pattern for stamping `cache_control` onto Anthropic content blocks; routing through `synth_file_message` means the document path inherits the same cache-control composition, so a tool-result PDF can still participate in Anthropic prompt caching.
- **Bedrock Converse parallel change at `:3977-4042` is a real fix, not just defensive widening.** Bedrock's Converse API has its own document-content shape (different from Anthropic's), and the parallel test suite at `test_converse_transformation.py:4146+` (+221 lines) confirms the Bedrock-side document-block construction is exercised end-to-end. Without this mirror change, callers who hit Bedrock instead of Anthropic-direct would see the same wire-rejection bug.
- **407 lines of tests for ~80 lines of production code.** The test-to-code ratio reflects that this is a translation layer where every input shape (mime type, URL form, content-list position, cache-control, missing-fields) needs an explicit case to lock the behavior. That's the right ratio for a translation module.

## Nits (non-blocking)

1. **`_ANTHROPIC_DOCUMENT_BASE64_MEDIA_TYPES` is a duplicate source of truth alongside `select_anthropic_content_block_type_for_file`.** The comment at `:1666-1668` correctly names the dependency, but a future maintainer who adds `text/csv` to the upstream selector and forgets to update this constant gets a silent regression where the new mime type falls through to the image path again. Either import the canonical set from the same module that defines `select_anthropic_content_block_type_for_file`, or add a unit test that asserts `_ANTHROPIC_DOCUMENT_BASE64_MEDIA_TYPES == set(<canonical-source>)` so drift is caught at test time.

2. **The regex at `:1670` is `r"data:([^;,]+)"` which is permissive about whitespace.** A URL like `"data: application/pdf;base64,..."` (note the leading space) would extract `" application/pdf"` and miss the set membership check, falling through to the image path. Worth a `.strip()` on the captured group, or tightening the regex to `r"data:([^\s;,]+)"`.

3. **No URL-length guard before regex.** A malicious or buggy caller could send a megabyte-long fake data URI; `re.match` is bounded but the surrounding `convert_to_anthropic_tool_result` does not appear to gate the URL size. Realistically the request body limit would catch it upstream, but a sniff-only-the-prefix optimization (`url[:64]`) would be cheap and bound the regex cost.

4. **The synthesized `ChatCompletionFileObject` (`synth_file_message` at `:1764+`) is built from positional/keyword data extracted from the image_url content** — verify the synthetic object's `file` dict has all required fields (`file_data`, `filename`, `format`) so the downstream document-translation path doesn't `KeyError` on legitimate inputs missing optional fields. The 186-line test suite likely covers this, but worth a quick spot-check for the case "data URI with no filename hint" (e.g. raw `data:application/pdf;base64,...` with no inline `filename=`).

5. **Bedrock test count (221) exceeds Anthropic test count (186)** which is unusual for a fix that originated on the Anthropic side. Skim the Bedrock test file to confirm it isn't carrying behavior assertions that contradict the Anthropic side (cache-control stamping order, content-list position when mixed with text content, etc.); divergence between the two test suites is the early-warning sign for divergence between the two production paths.
