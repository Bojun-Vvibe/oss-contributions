# BerriAI/litellm #26958 — fix: parse multipart user_config JSON

- **PR**: https://github.com/BerriAI/litellm/pull/26958
- **Head SHA**: `421b9e2d00cb203b9a820b0789b974b732ce5724`
- **Files reviewed**: `litellm/proxy/common_utils/http_parsing_utils.py`, `tests/test_litellm/proxy/common_utils/test_http_parsing_utils.py`
- **Date**: 2026-05-01 (drip-230)

## Context

The LiteLLM proxy accepts requests as either JSON (`application/json`) or multipart
(`multipart/form-data`, used for file-upload endpoints — image edits, audio
transcription, etc.). The shared body parser at `_read_request_body` had a special-case
for multipart that decoded `metadata` from JSON-string back to dict, but missed:

- `litellm_metadata` — the proxy's per-request bookkeeping field (request IDs, user
  IDs, anything `route_request` needs for routing/accounting).
- `user_config` — the **load-bearing** field that `route_request` splats into
  `Router(**user_config)` to construct a per-request router. With multipart the field
  arrives as the raw JSON string `'{"model_list": [...]}'`, which then becomes
  `Router(**"{...}")` — `TypeError`, request 500s.
- `tags` — JSON-list tags that JSON requests already supported but multipart didn't, so
  the same caller code worked for `chat/completions` JSON and silently broke for
  `/audio/transcriptions` multipart.

Fixes the user-reported failure mode in #26707.

## Diff

`http_parsing_utils.py:16` adds a tuple of fields that should always be JSON-decoded
when present as strings:

```python
_JSON_OBJECT_FORM_FIELDS = ("metadata", "litellm_metadata", "user_config")
```

`http_parsing_utils.py:41-56`:

```diff
  if "form" in content_type:
-     parsed_body = dict(await request.form())
-     if "metadata" in parsed_body and isinstance(parsed_body["metadata"], str):
-         parsed_body["metadata"] = json.loads(parsed_body["metadata"])
+     parsed_body: Dict[str, Any] = dict(await request.form())
+     for field in _JSON_OBJECT_FORM_FIELDS:
+         if field in parsed_body and isinstance(parsed_body[field], str):
+             parsed_body[field] = json.loads(parsed_body[field])
+     if "tags" in parsed_body and isinstance(parsed_body["tags"], str):
+         stripped_tags = parsed_body["tags"].strip()
+         if stripped_tags.startswith("["):
+             try:
+                 parsed_tags = json.loads(stripped_tags)
+             except json.JSONDecodeError:
+                 parsed_tags = None
+             if isinstance(parsed_tags, list):
+                 parsed_body["tags"] = parsed_tags
```

## Observations

1. **The two paths have correctly different error semantics.** For
   `_JSON_OBJECT_FORM_FIELDS` (`metadata` / `litellm_metadata` / `user_config`),
   `json.loads(...)` is **not** wrapped in try/except — invalid JSON in those fields
   raises `JSONDecodeError`, propagates as a 4xx, and the operator sees the error.
   That's the right call for `user_config` because a silent-fallback to a string would
   make `Router(**"{...}")` fail later with a much harder-to-diagnose `TypeError` deep in
   the routing stack. The new
   `test_form_data_with_invalid_json_user_config` at `:218-232` pins this contract
   with `pytest.raises(json.JSONDecodeError)`.

2. **For `tags`, the path is correctly conservative.** Tags can legitimately be a single
   string (`tags=production`), a comma-list (most callers), or a JSON list. The PR
   detects "looks JSON-ish" via `stripped_tags.startswith("[")` and only attempts
   parse when that gate fires; on parse failure or non-list result, the field is
   left untouched. The two regression tests at `:236-272`
   (`test_form_data_with_plain_string_tags` and
   `test_form_data_with_malformed_json_like_tags`) cover both the no-touch and
   parse-fail-no-touch paths. Right behavior — silently re-keeping `"production"` as
   `"production"` is what the existing JSON-shape callers expect; raising on `"[production"`
   would break callers who happen to use a tag literal that starts with `[`.

3. **`_JSON_OBJECT_FORM_FIELDS` as a module-level tuple** is the right shape: it's
   small, immutable, and the next "we missed another field" fix is a one-line tuple
   extension. Better than four parallel `if "foo" in parsed_body and isinstance(...)`
   blocks.

4. **`Dict[str, Any]` annotation at `:43`** removes the inferred-as-`Dict[str, str]`
   issue that crops up after the `for field in ...` loop reassigns string values to dicts —
   without the annotation mypy would type-error on the later `parsed_body["user_config"]["model_list"]`
   access in `route_request`. Small but load-bearing for type-cleanliness.

## Risks / nits

- **`json.loads` allows trailing whitespace + leading whitespace by default**, so
  `'  {"k": "v"}  '` parses fine. But it does not allow JSON5/comment syntax — if any
  caller is sending Python-`repr`-style dicts (`{'k': 'v'}` with single quotes), this
  will newly raise where it previously silently passed through unchanged. Worth
  surfacing in the changelog as a "stricter contract for `user_config` / `metadata` /
  `litellm_metadata` over multipart" note.
- **Tag-list parse uses only `startswith("[")`**, not `startswith("[") and endswith("]")`.
  A legitimate single-string tag of `"[draft] production"` would match the gate and try
  to JSON-parse, fail, and silently fall back to the original string — fine outcome but
  one wasted parse per request for that pattern. A two-character gate
  (`startswith("[") and endswith("]")`) would skip the parse attempt cheaply.
- **Loop variable `field`** at `:45-47` shadows nothing here, but a stricter linter would
  flag the same identifier appearing in `_JSON_OBJECT_FORM_FIELDS = ("metadata", ...)`
  comprehension suggestions; not real, just style.
- The pre-existing `F401, F601` ruff findings in the test file — the PR description
  flags this; agreed they should be a separate cleanup PR rather than mixed into this one.

## Verdict

merge-after-nits
