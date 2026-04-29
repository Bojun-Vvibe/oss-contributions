# PR #26723 — fix: decode multipart user_config JSON

- **Repo:** BerriAI/litellm
- **Link:** https://github.com/BerriAI/litellm/pull/26723
- **Author:** Genmin
- **State:** OPEN
- **Head SHA:** `c2795f2f81dca1b1dba818e3904e73108d7e8084`
- **Files:** `litellm/proxy/common_utils/http_parsing_utils.py` (+12/-2), `tests/test_litellm/proxy/common_utils/test_http_parsing_utils.py` (+82/-0)

Closes #26707.

## Context

When a request comes in as `multipart/form-data` (e.g. image-edit endpoints uploading an image file alongside JSON-encoded routing fields), `request.form()` returns each field as a string. Routing code downstream (per-request `user_config`, `tags`-based routing, request `litellm_metadata`) expects these to be `dict`/`list`, not raw JSON strings. The previous loader decoded `metadata` and *only* `metadata`:

```python
if "metadata" in parsed_body and isinstance(parsed_body["metadata"], str):
    parsed_body["metadata"] = json.loads(parsed_body["metadata"])
```

So `user_config` arrived as a string, downstream routing happily used it as a `model` selector or skipped it entirely, and per-request routing silently broke for any multipart endpoint. This is the canonical "JSON-body and form-body code paths drifted apart" bug.

## What changed

`http_parsing_utils.py:15` introduces a module-level constant:

```python
_JSON_STRING_FORM_FIELDS = ("metadata", "litellm_metadata", "user_config")
```

The original two-line decode site is replaced by a single helper call (`http_parsing_utils.py:44`):

```python
_decode_json_string_form_fields(parsed_body=parsed_body)
```

The helper at `http_parsing_utils.py:100-108`:

```python
def _decode_json_string_form_fields(parsed_body: Dict[str, Any]) -> None:
    for field_name in _JSON_STRING_FORM_FIELDS:
        if field_name in parsed_body and isinstance(parsed_body[field_name], str):
            parsed_body[field_name] = json.loads(parsed_body[field_name])

    tags = parsed_body.get("tags")
    if isinstance(tags, str) and tags.lstrip().startswith("["):
        parsed_body["tags"] = json.loads(tags)
```

Two design decisions worth pulling out:

1. The first three fields (`metadata`, `litellm_metadata`, `user_config`) get unconditional JSON decoding when they're strings. If `json.loads` fails, the exception propagates and the caller gets a 4xx — see test `test_form_data_with_invalid_json_user_config` which `pytest.raises(json.JSONDecodeError)`. This is **fail-loud**, which is the right call here: silently swallowing a malformed `user_config` would leave the request routed against partially-applied configuration.

2. `tags` gets a *gated* decode: only when the string starts with `[` (after `lstrip()`). This is the correctness-preserving carve-out — `tags` is the one field that has a legitimate single-string usage (`tags=image-edit`) as well as a JSON-list usage (`tags=["image-edit","per-request"]`). The `[`-prefix sniff distinguishes them without breaking existing `tags=single-string` callers, and `test_form_data_with_string_tag_is_preserved` pins that contract.

## Design analysis

This is a tight, scope-correct fix. Three observations:

1. **Why a module-level tuple, not an enum or registry?** A `tuple` of three strings is the cheapest possible structure for a "decode if present and string" loop. There's no semantic distinction between the three fields' decode behavior — they're all "JSON object" — so an enum would be overkill. The next field added is one tuple-element edit, which is exactly what you want for a list with strict additive growth.

2. **`tags` is special-cased rather than added to the tuple.** Right call: `tags` has the dual-shape contract (string OR JSON list), the other three don't. Mixing `tags` into the tuple loop would either break the string-tag contract or require a per-field branch that erodes the loop's simplicity.

3. **Test coverage is exactly the four cells you'd want.** Happy path (`test_form_data_with_json_user_config_and_tags`), preserved-legacy-behavior (`test_form_data_with_string_tag_is_preserved`), invalid JSON in the new field (`test_form_data_with_invalid_json_user_config`), and the existing `test_form_data_with_invalid_json_metadata` is preserved unchanged so the `metadata` regression stays pinned.

## Risks

1. **`_JSON_STRING_FORM_FIELDS` is module-private.** The leading underscore says "don't import this." Good — but if a downstream proxy plugin wants to add its own field, the current shape forces a fork. Worth a comment explaining the intentional scope ("public proxy fields only; plugin fields belong in their own decoder").

2. **`tags` gate sniffs only `[`.** `JSON arrays start with [` is correct, but a tag value like `"["` (single-character literal that happens to be a square bracket) would be incorrectly routed through `json.loads` and raise. Vanishingly unlikely in practice but worth a one-line comment near the `lstrip().startswith("[")` clarifying the assumption.

3. **`litellm_metadata` decoding is new behavior.** The old code only decoded `metadata`. If any caller previously passed `litellm_metadata` as a JSON-string in multipart and was relying on it being received as a string downstream (and then doing its own `json.loads`), this PR will now produce a `dict` where they expected a `str`, causing a `TypeError`. Probably nobody — the field name is internal — but worth a release-note callout.

4. **No content-length / DoS guard.** A multipart `user_config` field with a 100MB JSON string would happily reach `json.loads` and OOM. Pre-existing issue (the same is true for the old `metadata` decode), but the surface is now wider. Out of scope for this PR; worth a follow-up issue.

## Verdict

**Verdict:** merge-after-nits

Right shape, right scope, right tests. Nits:
- One-line comment near `_JSON_STRING_FORM_FIELDS` documenting the "proxy fields only, plugin fields fork their own decoder" intent.
- One-line comment near the `tags` gate clarifying the `[`-prefix-sniff assumption.
- Release-note callout for `litellm_metadata` now being JSON-decoded in multipart (newly-decoded field).

---

*Reviewed by drip-156.*
