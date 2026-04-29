# BerriAI/litellm #26759 — Anthropic document data-URI handling for tool results

- **PR:** https://github.com/BerriAI/litellm/pull/26759
- **Title (as posted):** "Litellm oss staging" (the title is generic; the diff is the
  fix below)
- **Head SHA:** d8d1444da42a6abc50074aa890f477943ac37a8b
- **Files changed:** 4 files, +517 / −12
- **Verdict:** `merge-after-nits`

## What it does

Fixes a real wire-format bug in the OpenAI-→-Anthropic translation for tool-result
messages that contain a `data:` URI with a non-image mime (specifically
`application/pdf` and `text/plain`). Previously, every `image_url` content block in a
tool result was unconditionally wrapped into Anthropic's `type: "image"` block; if the
data URI was a PDF, Anthropic 400'd because images may not have media type
`application/pdf`. The PR adds a small predicate
`_is_anthropic_document_data_uri` (`prompt_templates/factory.py:1664-1675`) that
inspects the data-URI media type prefix and routes those payloads through the
document-block code path instead, by synthesizing a `ChatCompletionFileObject` and
delegating to the existing `select_anthropic_content_block_type_for_file` helper.

## What's good

- The whitelist `_ANTHROPIC_DOCUMENT_BASE64_MEDIA_TYPES = {"application/pdf",
  "text/plain"}` (`factory.py:1664`) matches the documented Anthropic accepted set
  — not a bigger guess. The accompanying inline comment explicitly says "routing
  other mimes here would produce a document block the API rejects" which is the
  right policy choice (fail-loud on the image path with a known error rather than
  silently degrade through the document path with a different rejection).
- The Union expansion to include `AnthropicMessagesDocumentParam` in both the
  `anthropic_content` and `anthropic_content_list` types (lines ~1715-1734) is the
  type-correct shape; without it mypy would have caught the same issue as a runtime
  TypeError.
- Tests added under `tests/test_litellm/.../test_litellm_core_utils_prompt_templates_factory.py`
  and `tests/test_litellm/llms/bedrock/chat/test_converse_transformation.py` —
  covering both the Anthropic and Bedrock-Converse paths.

## Nits / risks

1. PR title "Litellm oss staging" is non-descriptive — please rename to something
   like "fix(anthropic): route PDF/text data URIs in tool results to document
   blocks". A scan against the changelog later will not find this PR.
2. Regex `re.match(r"data:([^;,]+)", url)` (line 1672) accepts `data:application/pdf`
   without `;base64,` — if a caller constructs a malformed data URI lacking the
   encoding suffix, this predicate returns true but the downstream document-block
   builder may then fail in a less-obvious place. Worth tightening to require the
   `;base64,` segment or adding an explicit early-return on missing encoding.
3. The `force_base64=True` Bedrock path receives the same routing — confirm in PR body
   that Bedrock-Converse's `document` block also accepts `text/plain` and not just
   `application/pdf`, since the whitelist is shared.
4. No new entry in CHANGELOG.md for the bugfix; downstream users currently
   working around this with manual base64 → file-id conversion will want to know.

## What I learned

When two transports share a type-translation helper (here, image-vs-document), the
right defensive move is a small whitelist with an inline rationale — not a
permissive predicate that silently accepts new mimes the upstream API will reject
later.
