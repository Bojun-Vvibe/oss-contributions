# BerriAI/litellm #27103 — fix(azure): omit model from deployment image gen and image edit bodies

- PR: https://github.com/BerriAI/litellm/pull/27103
- Head SHA: `c53c71ad6641d3b3a70a6d8659157359d62c6b26`
- Diff size: +192 / -105 across 9 files

## Summary

Fix for upstream issue #26316: when calling Azure OpenAI image
generation/edit endpoints via the `/openai/deployments/{deployment}/...`
URL shape, sending `model` in the request body breaks some models
(notably gpt-image-2). The fix introduces a body sanitizer that strips
`model` *only* when both `images/generations` (or `images/edits`) and
`/openai/deployments/` are present in the URL — provider-style URLs
(`/providers/...` for FLUX on Azure AI) keep all keys.

## Citations

- `litellm/llms/azure/azure.py:136-150` — new
  `azure_deployment_image_generation_json_body` static helper. The
  URL gating (`"images/generations" in api_base and
  "/openai/deployments/" in api_base`) is a substring match. Works
  for the standard URL shape but a path component named e.g.
  `/foo/openai/deployments/bar/images/generations/baz` would also
  match. In practice the Azure URL space is constrained enough that
  this is fine; would be cleaner with a `urlparse` + path-prefix
  check, but not blocking.
- `litellm/llms/azure/azure.py:985-993, 1107-1115` — both async and
  sync `make_*_azure_httpx_request` are updated symmetrically. Good.
- `litellm/llms/azure/image_edit/transformation.py:11-23` —
  `azure_deployment_image_edit_form_data` mirrors the same pattern
  for multipart form data. The fact that it uses `request_url`
  (the same string passed to `get_complete_url`) is documented in
  the docstring — that's important because previously this kind of
  late-stage URL inspection was inconsistent across providers.
- `litellm/llms/base_llm/image_edit/transformation.py:105-113` —
  new abstract method `finalize_image_edit_multipart_data` with a
  no-op default. Backward compatible for all other providers. ✔
- `litellm/llms/custom_httpx/llm_http_handler.py:5582-5584` —
  call site invokes the finalizer after
  `transform_image_edit_request`. The data passed in is the
  *non-file* form fields only — confirm with maintainers that
  removing `model` from these fields doesn't break any provider
  that *requires* it as a hint (e.g. the FLUX path on Azure AI).
  The Azure subclass guards against that by URL match, but other
  providers inheriting the default get an identity transform, so
  they're safe.

## Risk

Low. The behavior change is gated behind a precise URL pattern that
matches only the affected request shape. Other Azure paths (chat,
embeddings, non-deployment image URLs) are untouched.

## Verdict

`merge-as-is` — narrowly scoped, symmetric across sync/async, and
the abstract-method addition is backward-compatible. The substring
URL check is a minor cleanliness nit, not a correctness issue.
