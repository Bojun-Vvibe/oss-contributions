# All-Hands-AI/OpenHands PR #14029 — skip Gemini full endpoint URL as base_url to prevent doubled path 404

- **PR**: https://github.com/All-Hands-AI/OpenHands/pull/14029
- **Author**: @StatPan
- **Head SHA**: `4e513b97937202828b39a35b1b5d46ae8a2af0e9`

## Summary

`openhands/utils/llm.py:131-138` — `get_provider_api_base()` calls
`litellm.get_api_base(model, {})`, which for `gemini/<model>` returns
the *fully resolved* request URL (e.g.
`https://generativelanguage.googleapis.com/v1beta/models/gemini-pro:generateContent`),
not a reusable base. Persisting that as `base_url` and feeding it
back into the next litellm call caused litellm to append the model
path a second time — producing a doubled URL like
`.../models/gemini-pro:generateContent/models/gemini-pro:generateContent`
and a 404.

Fix:

```python
if api_base:
    if ':generateContent' in api_base or ':streamGenerateContent' in api_base:
        return None
    return api_base
```

Detect the Gemini-specific suffix marker and return `None` so litellm
falls back to its own resolution path. Test in
`tests/unit/utils/test_llm_utils.py:111-124` is rewritten from "asserts
the URL contains `generativelanguage.googleapis.com`" to "asserts
`None`" across five Gemini variants (pro, 3.1-pro-preview, 3-flash-preview,
2.5-pro, 2.5-flash).

## Verdict: `merge-as-is`

Right diagnosis, minimal fix, regression test pinned to issue #14028.
The `:generateContent` / `:streamGenerateContent` suffix detection is
the canonical marker for Gemini's REST API style and won't false-positive
on other providers.

## Specific references

- `llm.py:132-137` — the inline comment is excellent (calls out the
  exact failure mode: "litellm to append the model path a second
  time"). Future maintainers reading `if return None` won't ask
  "why".
- `test_llm_utils.py:111-124` — the regression test docstring cites
  #14028 explicitly. Five model variants is appropriately defensive
  against future Gemini model name additions.

## Risks

1. **Marker drift.** If Google adds a third Gemini API verb
   (`:embedContent`? `:countTokens` is already there) and litellm
   starts returning *that* URL in `get_api_base`, the suffix list
   will need updating. Consider broadening to
   `re.search(r':\w+Content', api_base)` or matching on
   `'/models/' in api_base` (any path containing the model segment
   indicates a non-base URL).
2. **Test brittleness.** The test asserts `is None` for specific
   model names. If litellm's behavior for those models changes (e.g.
   it stops returning the full URL because it gets fixed upstream),
   the assertion will start failing for the *right* reason but the
   test name (`returns_none_not_model_specific_endpoint`) becomes
   misleading. Consider asserting the *invariant*: "if api_base
   contains `:generateContent`, we return None" with a parametrized
   test that mocks `litellm.get_api_base`.
3. **Other providers with the same pattern.** Anthropic Vertex,
   Bedrock pass-through, and some Azure endpoints also return
   model-specific URLs. Worth a `git grep ':generate' litellm/`
   audit to see if the same defensive check is needed elsewhere —
   could make this a generalized "URL looks like a request URL not a
   base URL" guard.

## Nits

- The two suffix strings could be tuple-extracted to a module-level
  constant `_GEMINI_REQUEST_URL_MARKERS = (':generateContent', ':streamGenerateContent')`
  to make the future addition (point 1) trivial.
