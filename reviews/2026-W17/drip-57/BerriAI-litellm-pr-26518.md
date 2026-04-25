# BerriAI/litellm #26518 — chore(auth): tighten clientside api_base handling

- URL: https://github.com/BerriAI/litellm/pull/26518
- Head SHA: `2e2e1cbf7161e9cc7cf38161ccac8ed7a18af273`
- Verdict: **merge-after-nits**

## What it does

Three coordinated hardening changes around how the proxy accepts and
forwards a caller-supplied `api_base` / `base_url`:

1. SSRF gate at the proxy boundary in
   `litellm/proxy/auth/auth_utils.py::check_complete_credentials`.
2. Drop admin-configured provider credentials when a request redirects
   `api_base` / `base_url`, in
   `litellm/router_utils/clientside_credential_handler.py`.
3. Defense-in-depth `safe_get` wrap on the Vertex AI batch retrieve
   path in `litellm/llms/vertex_ai/batches/handler.py`.

## Diff notes

- `auth_utils.py:53-95` — the original code returned `True` as soon as
  an `api_key` string was present. The new flow keeps that necessary
  check, then loops over `("api_base", "base_url")` and runs each
  through `validate_url`, raising `ValueError` with the offending
  field name on `SSRFError`. Important subtlety: the check is gated by
  the existing `litellm.user_url_validation` master toggle (no-op
  when off), so it can't break a deployment that has explicitly opted
  out.
- `clientside_credential_handler.py:14-58` — the `_admin_config_fields_to_clear_on_base_override`
  helper derives the field list from
  `CredentialLiteLLMParams.model_fields` (so a new provider field
  added there is gated automatically) and unions it with a fixed list
  of kwargs-only fields (`organization`, `extra_body`,
  `aws_session_token`, etc.). This is the right pattern — the typed
  schema is the source of truth, the kwargs list is the documented
  escape hatch.
- `vertex_ai/batches/handler.py:225-236` — straightforward wrap of
  `sync_handler.get(api_base, ...)` in `safe_get`. Comment correctly
  notes this is defense-in-depth; the proxy gate is the primary
  control.

## Nits / requested changes

- The unit tests in `tests/test_litellm/proxy/auth/test_auth_utils.py`
  cover the auth-gate side and the credential-clearing side, but the
  Vertex `safe_get` wrap has no direct test (e.g. mock
  `sync_handler.get` rejected for a metadata IP). Worth one
  `pytest.raises(SSRFError)` row.
- `validate_url` is called twice (once per field) when both fields are
  present. Not hot-path, but trivially deduppable.

## Why this verdict

Real, well-scoped SSRF hardening that closes a credible "client supplies
`api_key=anything` plus `api_base=http://169.254.169.254/...`" pivot.
Schema-derived field list is the correct design. Just want one Vertex
test before merge.
