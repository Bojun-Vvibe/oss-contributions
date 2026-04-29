# BerriAI/litellm#26764 — `fix(proxy): decode multipart user_config JSON`

- URL: https://github.com/BerriAI/litellm/pull/26764
- Head SHA: `b63721c45f28`
- Author: Kcstring
- Size: +69 / -2 across 2 files
- Fixes: #26707

## Summary

Multipart proxy requests previously only auto-decoded the `metadata`
form field as JSON. `user_config` (per-request override of the
router config) and `tags` arrays were left as raw strings, so a
client sending the canonical multipart form for image-edit calls
hit `TypeError: string indices must be integers` deep in routing.
This PR generalizes the field-by-field JSON decode and adds two
focused unit tests.

## Specific references

`litellm/proxy/common_utils/http_parsing_utils.py`:

- New helper `_parse_form_json_field(parsed_body, field_name, *, require_json=False)`
  at lines 13–22:

  ```py
  if not isinstance(value, str):
      return
  if require_json or value.lstrip().startswith(("{", "[")):
      parsed_body[field_name] = json.loads(value)
  ```

- Replacement at lines 49–54 (form branch of `_read_request_body`):

  ```py
  parsed_body = dict(await request.form())
  _parse_form_json_field(parsed_body, "metadata", require_json=True)
  _parse_form_json_field(parsed_body, "user_config")
  _parse_form_json_field(parsed_body, "tags")
  ```

`tests/test_litellm/proxy/common_utils/test_multipart_json_form_fields.py` (new):

- `test_form_data_with_json_user_config_and_tags` — verifies a
  full `user_config` model_list payload plus a JSON-array `tags`
  string both round-trip into native objects.
- `test_form_data_with_plain_string_tags_is_left_unchanged` —
  pins the back-compat carve-out for clients that send `tags`
  as a single bare string ("production").

## Design analysis

The `require_json=True` carve-out for `metadata` preserves the
old "always parse, raise on bad JSON" behavior, while the new
heuristic for `user_config`/`tags` only parses when the value
*looks* like JSON (`{` or `[` prefix after stripping). That's the
right shape: a client that sends `tags=production` keeps working,
a client that sends `tags=["a","b"]` gets the array.

Two minor concerns about the heuristic:

1. A legitimate `user_config` value that is a JSON *string*
   (e.g. literally `"foo"` with quotes) would not start with `{`
   or `[`, so it stays a string. Probably fine because
   `user_config` is documented to be an object, but worth a
   one-line comment in the helper.
2. `require_json=False` swallows the parse only by virtue of the
   prefix check. If the prefix matches but the body is malformed
   JSON (e.g. `tags=[broken`), `json.loads` will still raise.
   That's actually the correct fail-loud behavior, but the test
   suite doesn't pin it. A `test_invalid_json_tags_raises` would
   document that contract.

## Risks

- Low. The change is additive and the existing `metadata`
  behavior is preserved verbatim via `require_json=True`.
- Watch for any callers downstream of `_read_request_body` that
  previously called `json.loads(parsed_body["user_config"])`
  themselves; they now receive a dict and would crash. The diff
  doesn't show any such call sites, but a quick
  `rg 'parsed_body\["user_config"\]'` and `rg 'body\["user_config"\]'`
  in proxy code is the right confidence check.

## Verdict

`merge-after-nits`

## Suggested nits

- Add a comment on `_parse_form_json_field` explaining the prefix
  heuristic and the back-compat goal.
- Add a third test asserting malformed JSON raises (pins the
  fail-loud behavior).
- Run `rg 'parsed_body\["user_config"\]'` to confirm no
  downstream caller still does its own decode.
