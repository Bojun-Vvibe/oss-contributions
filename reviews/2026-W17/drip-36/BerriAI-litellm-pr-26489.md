# BerriAI/litellm #26489 — Redact credentials in vector-store list/info/update; per-store update gate

- **Repo**: BerriAI/litellm
- **PR**: [#26489](https://github.com/BerriAI/litellm/pull/26489)
- **Head SHA**: `44a244bf15a68f1a438a1d992bac6670670c25ec`
- **Author**: stuxf
- **State**: OPEN (+298 / -26)
- **Verdict**: `merge-after-nits`

## Context

`LiteLLM_ManagedVectorStore.litellm_params` is the in-DB blob that
holds upstream provider credentials — OpenAI `api_key`, AWS
`aws_access_key_id` / `aws_secret_access_key`, Vertex
`vertex_credentials`, Azure connection strings, etc. The management
endpoints `GET /vector_store/list`, `POST /vector_store/info`, and
`POST /vector_store/update` returned that blob verbatim to any
authenticated caller who could pass the premium-feature gate. That's
a credential-disclosure bug — anyone with a low-privilege virtual key
on the proxy could enumerate and exfiltrate every provider key
configured against any vector store. The PR also tightens the
`update` endpoint to require per-store access, not just "you have a
valid key".

## Design

Two complementary fixes in
`litellm/proxy/vector_store_endpoints/management_endpoints.py`:

1. **Credential redaction at the response boundary**. The new helper
   `_redact_sensitive_litellm_params` (lines 36-53 of the diff) walks
   the dict and replaces any key matching
   `_LITELLM_PARAMS_MASKER.is_sensitive_key(k)` with the
   `REDACTED_BY_LITELM_STRING` constant. Non-secret keys (`api_base`,
   `region`, `model`, `api_version`) survive — that's important so
   ops can still see what provider/region the store points at.
   Wired into `list_vector_stores` at line 90-94, `get_vector_store_info`
   at lines 103-105, and the update path at lines 124-127.

   The masker addition at
   `litellm/litellm_core_utils/sensitive_data_masker.py:21-23` is
   subtle but correct — it adds `"credentials"` (plural) to the
   sensitive-key list. The comment notes Vertex uses
   `vertex_credentials`, and the existing segment-exact match for
   `"credential"` (singular) didn't catch it. Without this addition
   Vertex keys would have *leaked through* the new redaction layer.

2. **Per-store authorization for update** (lines 115-118 and
   146-148). The new helper `_fetch_and_authorize_vector_store`
   (lines 56-80) does a single fetch + access-check + 404/403 split:
   `HTTPException(404)` on miss, `HTTPException(403)` on access
   denial. The update endpoint now goes through it, closing the
   "any authenticated caller can mutate any store" gap.

## Risks

- **Backward-compat**: any client that was *relying* on
  `litellm_params.api_key` round-tripping out of the list/info
  endpoints will break. That's the right break — those clients are
  the bug's symptom — but a release-note callout is warranted.
- **Constant name typo**: `REDACTED_BY_LITELM_STRING` (line 23, 48)
  is missing the second `L` — should presumably be `REDACTED_BY_LITELLM_STRING`.
  Worth checking whether the constant is already defined that way in
  `litellm/constants.py` or if this is a new typo'd constant. If the
  existing constant has the typo, this PR isn't the place to fix it,
  but a follow-up rename is worth filing.
- **`_LITELLM_PARAMS_MASKER` is module-level** (line 33) — fine, the
  masker is stateless after init, but worth confirming it isn't
  populated lazily from config in a way that would race with first
  request after import.
- **`isinstance(litellm_params, dict)` at line 44** is defensive but
  silently returns the original (possibly-non-dict) value if the
  shape is wrong, which means a malformed row could still leak. A
  warn-log on the unexpected-shape branch would catch DB drift.

## Verdict

`merge-after-nits` — security-positive, correct on the wire, the only
items are constant-name typo confirmation and a logging nit. Ship.
