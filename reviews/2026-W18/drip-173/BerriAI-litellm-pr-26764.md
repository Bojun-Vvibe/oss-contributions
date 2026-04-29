# BerriAI/litellm #26764 — decode multipart user_config / tags JSON fields

- **PR:** https://github.com/BerriAI/litellm/pull/26764
- **Title:** `fix(proxy): decode multipart user_config JSON`
- **Author:** Kcstring
- **Head SHA:** b63721c45f2801c9ec4bb92a89b45e7e8c413fd6
- **Files changed:** 2 (`litellm/proxy/common_utils/http_parsing_utils.py`,
  new `tests/test_litellm/proxy/common_utils/test_multipart_json_form_fields.py`),
  +69 / −2
- **Verdict:** `merge-after-nits`

## What it does

Generalizes the existing `metadata`-only JSON-string decode path inside
`_read_request_body` (`http_parsing_utils.py:39-49` pre-change) to also
decode `user_config` and `tags` fields when they arrive as a JSON-shaped
string in a `multipart/form-data` body. Previously, sending an image-edit
request as multipart with a serialized `user_config` JSON string left
that field as a string downstream — breaking provider routing.

The new helper `_parse_form_json_field` (`http_parsing_utils.py:16-22`):

```python
def _parse_form_json_field(
    parsed_body: Dict[str, Any], field_name: str, *, require_json: bool = False
) -> None:
    value = parsed_body.get(field_name)
    if not isinstance(value, str):
        return
    if require_json or value.lstrip().startswith(("{", "[")):
        parsed_body[field_name] = json.loads(value)
```

Then call sites at lines ~52-54:

```python
_parse_form_json_field(parsed_body, "metadata", require_json=True)
_parse_form_json_field(parsed_body, "user_config")
_parse_form_json_field(parsed_body, "tags")
```

So `metadata` keeps its previous strict behavior (always `json.loads`),
while `user_config` / `tags` only get decoded if they look JSON-shaped
(`{`/`[` prefix). The new test
`test_form_data_with_plain_string_tags_is_left_unchanged` asserts the
heuristic: a plain string `"production"` for `tags` is preserved, not
JSON-decoded into an error.

## What's good

- Correct behavior split: `metadata` is documented to be JSON-only so
  `require_json=True` preserves the loud failure mode for malformed
  metadata, while `tags`/`user_config` accept both shapes — matching
  how those fields are actually used in the wild.
- The `value.lstrip().startswith(("{", "["))` heuristic is the right
  one: cheap, safe, and matches what every JSON-or-string sniffer in
  the OpenAI/HuggingFace ecosystems uses.
- The two new tests in
  `test_multipart_json_form_fields.py:9-56` cover both the positive
  (decode `user_config` + `tags`) and the regression-guarding negative
  (`tags="production"` left untouched) paths. They mock at the
  `request.form()` level which is the right boundary.
- Backward compatible: a caller still sending a dict directly in JSON
  body hits the existing `json.loads(body)` branch and never enters
  the form path.

## Nits / risks

- `json.loads` raises `JSONDecodeError` if the field starts with `{`
  but isn't actually valid JSON (e.g. `"{not really json"`). For
  `metadata` this is the established loud-fail behavior; for
  `user_config` and `tags` the heuristic-trigger means a user can
  now hit a 500 by passing `tags="{oops"` where previously they'd
  have gotten the string through. Catching `JSONDecodeError` and
  leaving the value as a string for the non-strict path would be
  more user-friendly, e.g.:

  ```python
  if require_json or value.lstrip().startswith(("{", "[")):
      try:
          parsed_body[field_name] = json.loads(value)
      except json.JSONDecodeError:
          if require_json:
              raise
          # leave as string for opportunistic decode
  ```

- Empty string handling: `value.lstrip()` on `""` returns `""`, which
  doesn't start with `{` or `[`, so `""` survives as `""`. Good. But
  `value=" "` will likewise survive — confirm that's desired for
  `user_config` (probably yes, but worth a one-line test).
- The PR title is `fix(proxy): decode multipart user_config JSON` but
  the diff also changes `tags` behavior. Title could read
  `decode multipart user_config and tags JSON fields` for accurate
  blame.
- This is a near-duplicate of #26723 (also "decode multipart
  user_config JSON" by a different author). Maintainers should pick
  one and close the other; this PR has the more general helper and
  better tests, so prefer this one and close #26723.

## Verdict rationale

`merge-after-nits`: the refactor is clean and the tests are good; the
JSONDecodeError handling for the non-strict path is a real edge case
that should be addressed before merge to avoid trading a silent-failure
bug for a noisy-failure bug.
