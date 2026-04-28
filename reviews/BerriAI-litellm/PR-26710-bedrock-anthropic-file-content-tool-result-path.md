# BerriAI/litellm#26710 — fix(bedrock, anthropic): translate OpenAI file content on tool-result path

- **Repo:** [BerriAI/litellm](https://github.com/BerriAI/litellm)
- **PR:** [#26710](https://github.com/BerriAI/litellm/pull/26710)
- **Head SHA:** `5b5363cd5447d898b1c981beb997436e6b167cf1`
- **Size:** +561 / -12 across 6 files (3 production, 3 test files)
- **State:** OPEN
- **Fixes:** #24641. **Supersedes:** #24646.

## Context

Closes a real correctness bug on the **tool-result path** for both Bedrock
Converse and direct Anthropic. The user-message path already correctly
translated `{"type": "file", "file": {"file_data": "data:application/pdf;..."}}`
into a document block via `anthropic_process_openai_file_message`. The
tool-result path didn't — so when a tool emitted a PDF as part of its result,
the translation produced an empty `toolResult.content: []` for Bedrock and a
zero-document tool_result content for Anthropic. The model then saw "the tool
returned nothing" and responded accordingly. Silent data drop, no error.

The PR also fixes a *second*, more subtle bug: when a caller wraps a PDF as
an `image_url` data URI (`data:application/pdf;base64,...`), Anthropic
explicitly rejects it as `image` (mime mismatch) and Bedrock's
`BedrockImageProcessor.process_image_sync` correctly returns a
`{"document": ...}` block — but the tool-result wrapper at
`factory.py:_convert_to_bedrock_tool_call_result` only had an `if "image" in
_block` arm. Documents fell on the floor.

## Design analysis

### 1. Anthropic path — `factory.py:1664-1672`

The mime-prefix predicate is the right shape:

```python
def _is_anthropic_document_data_uri(url: str) -> bool:
    match = re.match(r"data:([^;,]+)", url)
    if not match:
        return False
    mime_type = match.group(1)
    return mime_type.startswith("application/") or mime_type.startswith("text/")
```

This is conservative — it routes anything `application/*` or `text/*` to the
document path, which matches Anthropic's documented document-block media-type
acceptance. PNG/JPEG/etc. (mime `image/*`) still flows through the
existing `create_anthropic_image_param` branch. Good split, no overlap.

The branching in `convert_to_anthropic_tool_result` (factory.py:1742-1788) is
correct:

- If the content is `image_url` and the URL is a data URI with an
  application/* or text/* mime type → synthesize a `ChatCompletionFileObject`
  shape and route through `anthropic_process_openai_file_message` (which is
  the same helper the user-message path uses). One source of truth for the
  document-block construction.
- Otherwise (real image data URI or http(s) URL) → existing
  `create_anthropic_image_param` path.
- New: `elif content["type"] == "file"` → also goes through
  `anthropic_process_openai_file_message`.

The `add_cache_control_to_content` calls are preserved on both new branches,
so callers using prompt caching with PDFs continue to get cache-control
metadata applied. That's easy to forget.

### 2. Bedrock path — `factory.py:4040-4060`

Two new branches in `_convert_to_bedrock_tool_call_result`:

```python
elif "document" in _block:
    tool_result_content_blocks.append(
        BedrockToolResultContentBlock(document=_block["document"])
    )
elif content["type"] == "file":
    file_obj = content.get("file") or {}
    file_data = file_obj.get("file_data")
    if isinstance(file_data, str):
        _file_block: BedrockContentBlock = (
            BedrockImageProcessor.process_image_sync(image_url=file_data)
        )
        if "document" in _file_block:
            tool_result_content_blocks.append(
                BedrockToolResultContentBlock(document=_file_block["document"])
            )
        elif "image" in _file_block:
            tool_result_content_blocks.append(
                BedrockToolResultContentBlock(image=_file_block["image"])
            )
```

The first new branch (the `elif "document" in _block:` line) is the
image-url-with-document-mime regression fix. The second new branch is the
`type: "file"` path that was simply not handled.

Reusing `BedrockImageProcessor.process_image_sync` for the file path is the
right choice — it already understands base64 data URIs and dispatches to
either `image` or `document` correctly. The wrapper only needs to read the
output discriminant and append the right `BedrockToolResultContentBlock`
variant.

### 3. Type widening — `anthropic.py:332-339`

`AnthropicMessagesToolResultParam.content`'s union was widened to include
`AnthropicMessagesDocumentParam`. That's the type-system reflection of the
runtime change and keeps mypy/pyright callers honest about what shapes they
might receive. No behavior change.

### 4. Test coverage — six new tests

The test layout is good: each direction (Anthropic, Bedrock) gets three
tests:

- `..._openai_file_pdf_becomes_document` — primary positive case
  (`type: "file"`).
- `..._image_url_pdf_data_uri_becomes_document` — regression for the
  silent-drop bug on the `image_url`-with-PDF-mime path.
- `..._image_url_png_still_becomes_image` — locks the existing
  PNG-stays-image behavior so the new branching doesn't accidentally
  funnel images into documents.

The Bedrock tests assert specific structural details (`block["document"]["format"] == "pdf"`,
`block["document"]["name"].startswith("DocumentPDFmessages_")`,
`block["document"]["source"]["bytes"] == pdf_b64`). That's the right depth:
the format / name fields are what Bedrock actually validates server-side, so
asserting them here catches translation bugs before they hit the Bedrock API.

## Risks / nits

1. **`re.match(r"data:([^;,]+)", url)` is loose on edge cases.** A URL like
   `data:application` (no `;` or `,`) would match `application` and route
   to document. In practice, all valid data URIs have `;base64,` or `,`,
   so this is fine — but `r"data:([^;,]+)[;,]"` would tighten the match
   without rejecting anything legitimate.
2. **`anthropic_process_openai_file_message` is called with a synthesized
   `ChatCompletionFileObject` that omits `filename`** for the
   `image_url`-with-PDF-data-URI branch (factory.py:1755-1758). If the
   downstream document-block builder uses `filename` for anything other
   than a fallback name, the synthesized path will produce slightly
   different output than the native `type: "file"` path. The test
   `test_anthropic_tool_result_image_url_pdf_data_uri_becomes_document`
   only asserts media_type and data, not name — so we're not currently
   pinning name parity. If parity matters, add an assertion; if not,
   document the asymmetry.
3. **No test for non-PDF `application/*` mimes** (e.g., `application/json`,
   `application/octet-stream`). The predicate accepts them; it'd be worth
   one parametrized test confirming Anthropic / Bedrock translation handles
   (or sensibly rejects) those — especially `application/octet-stream`,
   which neither provider documents acceptance for.
4. **No test for `text/*` mimes.** The predicate accepts `text/plain`,
   `text/csv`, etc. Anthropic's docs cover `text/*` for document blocks;
   a smoke test would lock the routing.
5. **PR checkbox: Greptile review confidence ≥4/5 not yet completed.** Per
   the litellm contributor checklist, that's a maintainer prereq. Worth
   noting in the merge sequence.

## Verdict

**merge-after-nits.** This is a high-quality bug fix with the right
abstractions reused (no duplicate document-block construction), six focused
tests covering both the primary `type: "file"` path and the regression
`image_url`-with-PDF-mime path, and conservative mime predicates. The nits
are coverage-extension and edge-case-tightening, not correctness blockers
for the documented PDF use case.

## What I learned

- When two code paths translate the same source shape and only one was kept
  in sync with newly-added content types, the silent-drop bug surfaces as
  "the model acts like the tool returned nothing." That's hard to debug
  from logs alone — it looks like a model behavior issue, not a translation
  issue.
- Reusing the same helper (`anthropic_process_openai_file_message`) on
  both the user-message and tool-result paths is the correct dedup. The
  alternative (copy-paste with a different prefix) would have left the
  bug latent for the next mime-type extension.
- `BedrockImageProcessor.process_image_sync` returning a tagged-union
  `{"image": ...} | {"document": ...}` is a useful pattern — but every
  consumer needs to handle *both* arms. Discriminated-union exhaustiveness
  is the kind of thing a Python type checker can catch with `Literal`-keyed
  TypedDicts; worth a follow-up.
