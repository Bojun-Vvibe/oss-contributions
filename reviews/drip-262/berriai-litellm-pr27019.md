---
pr: https://github.com/BerriAI/litellm/pull/27019
head_sha: 8791f634def4cfff707ada6ca15af020f5388247
author: stuxf
additions: 997
deletions: 200
files_touched: 18
---

# Scope

Security hardening of cloud-storage-backed file paths for Vertex AI,
Bedrock, and GCS bucket logging. Introduces a new shared module
`litellm/litellm_core_utils/cloud_storage_security.py` (+175) with
sanitization, prefix-validation, and URL-encoding helpers; reworks the
file retrieve/content/delete flows for Vertex/Bedrock to require that
caller-supplied file IDs resolve inside the configured bucket *and* a
managed prefix; and removes `gcs_bucket_name` /
`gcs_path_service_account` from the request-driven dynamic callback
params (these are now server-only). Adds 200+ lines of regression tests.
Linked tickets: VERIA-45, VERIA-59.

# Specific findings

- `litellm/litellm_core_utils/cloud_storage_security.py:79-93` (head
  `8791f63`) — `_validate_cloud_object_path` rejects empty names,
  absolute paths, control chars, and `.`/`..` segments, and disallows
  empty interior segments. Solid baseline. Consider also rejecting paths
  that contain a literal `\` (backslash) — `split_configured_cloud_bucket_name`
  rejects backslashes in the bucket part but `_validate_cloud_object_path`
  accepts them inside object paths, where some downstream tooling will
  treat them as separators.
- `litellm/litellm_core_utils/cloud_storage_security.py:139-174` —
  `validate_managed_cloud_file_id` gates by `(configured_bucket,
  managed_prefix)`. Good. The `allow_legacy_cloud_file_ids` escape
  hatch (147, 165) is keyed off `_litellm_internal_model_credentials`
  (a `MappingProxyType`) per `should_allow_legacy_cloud_file_ids`
  (115-132). Two follow-up concerns:
  - The flag value is parsed from a string with
    `value.strip().lower() in {"1","true","yes","on"}` — fine, but
    please document that this is a *trusted-credential-only* opt-in and
    not user-controllable.
  - Rejection still requires the path to start with
    `f"{configured_prefix.rstrip('/')}/"` (162-164), which is good — but
    the early-return at 165 also bypasses any further sanity check on
    the object path. Confirm that `_validate_cloud_object_path` was
    already enforced at 152 before the legacy branch is taken (it is —
    line 152). Worth a code comment to make that ordering explicit.
- `litellm/integrations/gcs_bucket/gcs_bucket.py:340-344` —
  `_get_object_name` previously honored `_metadata["gcs_log_id"]` as
  the *exact* object name, which let any caller able to influence
  metadata pin a known path (and thus collide / overwrite). The new
  code passes it through `sanitize_cloud_object_component` and prefixes
  with `current_date/custom-{uuid}-`, removing both the path-injection
  risk and the predictable overwrite vector. Good change. Ensure
  downstream consumers reading `gcs_log_id` for retrieval (if any) are
  updated — the diff drops the `quote(object_name, safe="")` at
  gcs_bucket.py:370 in favor of relying on `download_gcs_object` to
  encode internally.
- `litellm/integrations/gcs_bucket/gcs_bucket_base.py:255,295,346` —
  `encode_gcs_object_name_for_url` is now applied at the three GCS
  HTTP boundaries (download/delete/upload). The helper does
  `quote(unquote(name), safe="")` (cloud_storage_security.py:101) — i.e.
  it idempotently re-encodes — which is the right shape but means a
  caller who somehow gets a literal `%XX` byte sequence into an object
  name will see it decoded once. For names that came from
  `sanitize_cloud_object_component` this can't happen (the regex strips
  anything outside `A-Za-z0-9._-`), but for names that survive the
  legacy-file-ID escape hatch, please confirm that an attacker can't
  smuggle a `..` via percent-encoding (unquote then validate ordering
  matters here).
- `litellm/litellm_core_utils/initialize_dynamic_callback_params.py:40-48,55-58`
  — moves `gcs_bucket_name` and `gcs_path_service_account` out of
  `_supported_callback_params` and into a new
  `_request_blocked_callback_params` set, then explicitly skips them in
  both the kwargs and metadata loops. This is the right fix for the
  "client requests can choose GCS logging storage destinations through
  callback params" issue called out in the PR body. Verify there isn't
  a third loop somewhere (e.g. in headers-based dynamic params)
  that still reads these names.
- `tests/test_litellm/llms/bedrock/files/test_bedrock_files_handler.py`
  (+206 lines) — substantial regression cover. Strong.

# Suggested changes

1. Reject `\` in object paths inside `_validate_cloud_object_path` to
   match the bucket-name handling.
2. Add a short doc-comment to `validate_managed_cloud_file_id` clarifying
   the order of operations: scheme check → bucket match →
   `_validate_cloud_object_path` → managed-prefix check → optional legacy
   escape hatch.
3. Add a regression test for percent-encoded `..` smuggling in the
   legacy-file-ID branch (defense-in-depth — should already fail at
   `_validate_cloud_object_path` since it runs on the unquoted form,
   but pin it).
4. Document that `_litellm_internal_model_credentials.allow_legacy_cloud_file_ids`
   is a *trusted-config* opt-in only and must never come from end-user
   request bodies.

# Verdict

`merge-after-nits`

# Confidence

High — the threat model is clear, the new module has tight invariants,
test coverage looks proportionate to the blast radius, and the follow-up
items are clarifications/defense-in-depth rather than blockers.

# Time spent

~14 minutes.
