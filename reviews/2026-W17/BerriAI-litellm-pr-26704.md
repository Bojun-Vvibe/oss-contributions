# BerriAI/litellm PR #26704 — fix: support Anthropic image and document in token counter

- URL: https://github.com/BerriAI/litellm/pull/26704
- Head SHA: 357609364179
- Files: `litellm/litellm_core_utils/token_counter.py`, `tests/test_litellm/litellm_core_utils/test_token_counter.py`
- Verdict: **merge-as-is**

## Context

`_count_content_list` walks Anthropic-shaped content blocks and
tallies tokens per block type. The existing branches handled
`text`, `image_url` (the OpenAI-style block), `tool_use`,
`tool_result`, and `thinking` — but Anthropic's *native* content
blocks `{"type": "image", "source": {...}}` and
`{"type": "document", "source": {...}}` fell through to the
`else` arm, which raises `ValueError` and is then caught by the
outer `except Exception` and (per the surrounding code's
behavior) logs noise. Net effect: native Anthropic image/document
inputs spam the logs and produce unreliable token counts.

## Design analysis

Two new helpers cover the two block shapes:

`_count_anthropic_image` (lines 9-54) handles the three documented
`source.type` variants:

- `base64`: reconstructs the data URL and calls
  `calculate_img_tokens(data=data_url, mode="auto", ...)` — the
  same path the OpenAI `image_url` branch already uses, which
  guarantees consistent tokenization across the two source shapes.
- `url`: passes the URL through to `calculate_img_tokens`. Raises
  `ValueError` if `url` is missing — fail-loud is right here, an
  empty URL is a malformed request the caller should know about.
- `file` and unknown future types: returns `DEFAULT_IMAGE_TOKEN_COUNT`.
  The docstring spells out *why* — there's no fetchable payload
  to dimension and silently approximating is preferable to
  re-introducing the log-spam this PR fixes.

`_count_anthropic_document` (lines 57-79) handles the more
heterogeneous `document` block:

- Tokenizes `title` and `context` text fields with the model
  tokenizer.
- Adds `DEFAULT_IMAGE_TOKEN_COUNT` if `source` is present (the
  payload is opaque — typically a PDF — and not dimensionable).
- Explicitly skips `cache_control` and `citations` (configuration
  metadata, not prompt content). This list of skipped fields is
  documented in the docstring, which will save the next reader
  from grepping the Anthropic schema.

The dispatch in `_count_content_list` adds two clean
`elif c["type"] == "image"` / `"document"` arms before the
fallback. The fallback `ValueError` message is also updated to
list the new accepted types.

Test coverage in `test_token_counter.py` looks substantial (211+
lines added) and includes a real 1×1 PNG base64-encoded inline so
the base64 path is exercised end-to-end.

## Risks / suggestions

1. `_count_anthropic_image` raises `ValueError` for missing
   `source` or missing `url`. Confirm the surrounding `try/except`
   in `_count_content_list` (line 105+) treats these as the
   "abort with informative message" path, not as silent skip.
   The previous code raised in the same way, so behavior is
   preserved — just worth confirming.
2. Falling back to `DEFAULT_IMAGE_TOKEN_COUNT` for `source.type
   == "file"` may under- or over-count by a wide margin depending
   on the underlying file size. That's unavoidable without
   fetching the file, but consider exposing
   `use_default_image_token_count` so callers who *can* dimension
   the file (via a separate lookup) can override.
3. No end-to-end integration test against a real Anthropic
   response — the PR relies on shape-only fixtures. Probably
   fine given the helpers are small, but worth flagging.

## What I learned

Token counters for vendor-specific content shapes silently
under-count or over-count whenever a new block type lands in the
schema. The right defensive posture is to enumerate every
documented variant explicitly with a fail-loud `else` (so the
next vendor addition surfaces immediately) plus a documented
default for opaque payloads. This PR follows that pattern
cleanly, and the helper extraction makes each block type's
tokenization rules self-documenting.
