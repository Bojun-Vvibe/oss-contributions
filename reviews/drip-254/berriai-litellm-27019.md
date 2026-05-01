# BerriAI/litellm #27019 — fix(files): constrain cloud storage file paths (VERIA-45, VERIA-59)

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/27019
- **HEAD SHA:** `ba1188117d3ca5564d42825d6167dd6d37795a26`
- **Author:** stuxf
- **Verdict:** `merge-after-nits`

## What the diff does

Tightens cloud-storage-backed file paths for Vertex AI (GCS),
Bedrock (S3), and GCS logging — three vectors where the prior code
trusted a caller-supplied object identifier or callback parameter
and let it reach a server-credentialed cloud SDK call, opening up:

- **Tenant-controlled path traversal in retrieve/content/delete**
  via raw cloud object IDs (`gs://other-tenant-bucket/secrets.json`,
  `s3://configured-bucket/../../etc/secret.json`, etc.).
- **Tenant-controlled GCS log destination override** via callback
  params (`gcs_bucket_name`, `gcs_path_service_account` injected
  via request `kwargs` or `metadata`, redirecting the operator's
  configured logging bucket to a tenant-controlled one).
- **Tenant-controlled object naming** via `gcs_log_id` metadata
  used as a verbatim object path (slashes, `..`, control chars all
  preserved).

Files of note:

- `litellm/litellm_core_utils/cloud_storage_security.py` (new
  file, +140 lines) — the canonical primitive layer:
  - `sanitize_cloud_object_component(value, fallback="file")` at
    `:18-35` strips path separators (`\` → `/` then `posixpath.
    basename`), rejects `.`/`..`/empty, replaces control chars
    (`ord<32` or `==127`) with `_`, then runs the regex
    `_SAFE_OBJECT_COMPONENT_PATTERN = re.compile(r"[^A-Za-z0-9._-
    ]+")` to collapse anything outside the conservative allowlist
    to `_`, strips leading/trailing `._`, returns the fallback if
    nothing survives, and truncates to 255 chars (the GCS/S3
    object-key segment limit).
  - `sanitize_cloud_object_path(value, fallback="file")` at
    `:38-49` runs the component sanitizer per `/`-segment and
    rejoins.
  - `build_managed_cloud_object_name(prefix, filename, fallback_
    filename="file")` at `:52-58` is the constructor for new
    upload object keys: `f"{prefix}{uuid.uuid4().hex}-{safe_
    filename}"` — load-bearing primitive because every upload now
    carries a server-generated UUID prefix that's later used as
    the membership-test surface for retrieve/content/delete.
  - `_validate_cloud_object_path(object_name)` at `:61-69` is the
    structural-validity check (non-empty, not absolute, no
    control chars, no `.`/`..` segments).
  - `split_configured_cloud_bucket_name(bucket_name) -> (bucket,
    prefix)` at `:72-92` parses the operator-configured bucket
    string (e.g., `"my-bucket/logs/litellm"`) into a strict
    `(bucket, prefix)` tuple, rejecting URI schemes (`://`),
    query strings (`?`/`#`), control chars, and trailing `/`-only
    inputs.
  - `encode_gcs_object_name_for_url(object_name)` at `:95-96`
    does `quote(unquote(value), safe="")` — the
    decode-then-reencode is the right pattern for handling
    already-quoted input idempotently.
  - `validate_managed_cloud_file_id(file_id, scheme,
    configured_bucket_name, allowed_object_prefixes)` at
    `:103-140` is the load-bearing access-control primitive:
    decodes the URL-encoded file_id, asserts the scheme prefix
    (`gs://` / `s3://`), splits into `bucket/object`, asserts
    `bucket == configured_bucket`, validates the object path
    structurally, and asserts the object key starts with one of
    the LiteLLM-managed prefixes (optionally further constrained
    by the operator's configured sub-prefix). Returns the
    `(bucket_name, object_name)` tuple for the caller to pass to
    the cloud SDK.

- `litellm/llms/bedrock/files/handler.py:5-15` and
  `litellm/llms/bedrock/files/transformation.py` — bedrock file
  ops use `validate_managed_cloud_file_id` against the configured
  bucket and `BEDROCK_MANAGED_S3_PREFIXES = (BEDROCK_MANAGED_S3_
  BATCH_PREFIX, BEDROCK_MANAGED_S3_UPLOAD_PREFIX, BEDROCK_MANAGED_
  S3_OUTPUT_PREFIX)`. Upload object keys are now generated via
  `build_managed_cloud_object_name` so the post-upload file_id is
  guaranteed to round-trip through the validation gate.

- `litellm/llms/vertex_ai/files/handler.py` and
  `litellm/llms/vertex_ai/files/transformation.py` — same shape
  applied to vertex_ai with `VERTEX_AI_MANAGED_GCS_PREFIX =
  "litellm-vertex-files/"`.

- `litellm/integrations/gcs_bucket/gcs_bucket.py:339-346` — the
  `gcs_log_id` metadata is now sanitized through
  `sanitize_cloud_object_component` before being incorporated
  into the object name, *and* the assembled object name is now
  `f"{current_date}/custom-{uuid.uuid4().hex}-{safe_log_id}"`
  rather than the raw caller-supplied string. Sanitized empty
  string → fall back to the auto-generated default, which is the
  right fail-safe.

- `litellm/integrations/gcs_bucket/gcs_bucket.py:367-376` — the
  retrieve path no longer does `quote(object_name, safe="")`
  inline at the call site (now done inside `download_gcs_object`
  via `encode_gcs_object_name_for_url`), centralizing URL-encoding
  in one place.

- `litellm/integrations/gcs_bucket/gcs_bucket_base.py:133-144`
  swaps the inline `if "/" in bucket_name: bucket_name, prefix =
  bucket_name.split("/", 1)` for a call to `split_configured_
  cloud_bucket_name`, getting the URI-scheme-rejection /
  control-char-rejection / trailing-slash handling for free
  alongside the existing folder-in-bucket-name semantics.

- `litellm/litellm_core_utils/initialize_dynamic_callback_params.
  py:60-103` — adds `_request_blocked_callback_params = {"gcs_
  bucket_name", "gcs_path_service_account"}` (module-level
  frozenset-ish) and gates *both* the top-level kwargs loop
  (`:79-81`) *and* the metadata loop (`:96-98`) on `if param in
  _request_blocked_callback_params: continue`. Load-bearing
  detail: both code paths must skip these params, otherwise a
  caller setting `metadata.gcs_bucket_name = "evil-bucket"` would
  bypass the kwargs-level block via the metadata fallback. The
  comment "skip — these can only be configured server-side" would
  make the intent explicit.

- 6 test files updated/added: bedrock handler/transformation
  tests gain ~85+39 lines covering the validation gate
  (`+test_retrieve_with_external_bucket_rejected` etc.), vertex
  handler/transformation tests gain ~79+86 lines, gcs_bucket_
  base tests gain +43 lines covering the bucket-name splitting,
  and `test_initialize_dynamic_callback_params.py` gains +15
  lines pinning the request-blocked params.

## Why it's right

The architectural shape is correct: pulling the
sanitize/validate/encode primitives into a single
`cloud_storage_security.py` module gives each concern a named
function with a typed signature, and replacing inline trust-the-
caller patterns at the call sites with calls into that module makes
the security boundary explicit. The split between
*sanitize* (best-effort: strip dangerous chars, fall back to a
default) and *validate* (strict: raise `ValueError` on anything
suspicious) is the right two-tier shape — sanitization for caller-
supplied components that we're going to *write into* a managed
prefix (like `gcs_log_id` becoming part of the object name) and
validation for caller-supplied identifiers that we're going to
*read* (like a `file_id` that must round-trip back to a managed
upload).

The load-bearing security property — "any cloud SDK call that uses
a tenant-supplied object identifier first goes through
`validate_managed_cloud_file_id` against the operator's configured
bucket and the managed prefix set" — is now expressible as a
grep-able invariant rather than a code-review-time invariant.
Future callers adding new file ops can be checked by `grep -rn
'gs://\|s3://' litellm/llms/` and asserting every match either
goes through the validator or is justified.

The `build_managed_cloud_object_name` constructor producing
`f"{prefix}{uuid.uuid4().hex}-{safe_filename}"` is the right shape
because it makes the *prefix membership* the membership-test
surface for retrieve/content/delete: any file_id that doesn't start
with one of the managed prefixes can't have come from a LiteLLM
upload, so rejecting it is correct (no behavior loss for legitimate
users since legitimate users only got file_ids out of the managed
upload path).

The `_request_blocked_callback_params` carve-out at
`initialize_dynamic_callback_params.py:60-103` is the right shape
for a "config-only param that should never be flowable from
request" rule, and gating it identically at both the kwargs and
metadata loops closes the obvious bypass (set it on metadata if
kwargs is blocked).

The test surface is appropriately broad: 79 tests across 4 files
covering the validation gate, the bucket-prefix combination cases,
the request-blocked callback params on both kwargs and metadata
paths, and the gcs_log_id sanitization. The
`test_initialize_dynamic_callback_params.py` coverage of the
metadata-bypass-attempt is the most important one because that's
where the pre-fix code had the weakest invariant.

## Nits

1. **`sanitize_cloud_object_component`'s 255-char truncation at
   `:34`** — GCS object *key* limit is 1024 bytes total, S3 object
   key limit is also 1024 bytes; 255 is the segment limit for some
   filesystems but not for GCS/S3 keys. Truncating to 255 is
   conservative (won't break the API) but worth a comment that
   the bound is "intentionally below the cloud limit, sized for
   filename-safety" rather than "the cloud key limit." Otherwise
   a future "let me lift this to the GCS limit" change widens the
   trust surface.

2. **`encode_gcs_object_name_for_url(object_name)` at `:95-96`
   does `quote(unquote(value), safe="")`** — the decode-then-
   reencode is correct for idempotence on already-quoted input,
   but it also silently *changes* the meaning of an object name
   that legitimately contains `%XX`-shaped segments (e.g., if
   someone uploaded a file literally named `report%20final.pdf`,
   the `unquote` step turns it into `report final.pdf` and then
   re-encodes, producing a different on-disk lookup). This is a
   behavior change worth flagging in the PR description for
   operators who may have such files. Mitigation: strict mode
   that rejects already-encoded input, or a separate
   `encode_already_decoded_object_name_for_url` that doesn't do
   the unquote step.

3. **`validate_managed_cloud_file_id` error messages at `:124-
   140`** — `"file_id bucket does not match the configured
   storage bucket"` doesn't echo the bucket name back to the
   caller (good, no info leak), but `"file_id must reference a
   LiteLLM-managed storage object"` is generic enough that an
   attacker probing the prefix list can't enumerate it. That's
   the right call. However, the *operator* debugging a
   legitimate "why isn't my file_id being accepted" gets no
   signal about which prefix was expected — worth a
   `verbose_logger.debug` log line with the expected prefixes
   and the actual object_name prefix that didn't match, gated on
   debug-only so the prod path stays terse.

4. **`split_configured_cloud_bucket_name` at `:72-92` rejects
   `?` and `#`** in the bucket name (good — those are URI special
   characters and have no business in a cloud bucket name) but
   doesn't reject `@` (which can be present in some
   credentials-in-URL anti-pattern configurations). Worth adding
   `@` to the rejected character list, since legitimate cloud
   bucket names never contain it and seeing one is a strong
   signal of a config mistake or injection attempt.

5. **`_request_blocked_callback_params` at `:60-63`** is a Python
   `set` literal — fine, but a `frozenset` would express the
   "this is module-level immutable config" intent better and
   prevent a future caller from accidentally `.add()`-ing to it.
   Mechanical change.

6. **Two unrelated CVE fixes (VERIA-45 and VERIA-59) in one PR**
   — VERIA-45 is the file_id IDOR (validate_managed_cloud_file_id),
   VERIA-59 is the GCS-logging callback-param hijack
   (`_request_blocked_callback_params`). They share the
   `cloud_storage_security.py` infrastructure but are conceptually
   independent — a downstream security tracker triaging "which
   commits address VERIA-45 vs VERIA-59" has to do per-file
   archaeology to attribute. Worth splitting in two for future
   security-tracking ergonomics, though the shared infrastructure
   means the split would still keep the new module in one PR.

7. **The retired `from urllib.parse import quote` at
   `gcs_bucket.py:9`** is now done via the new helper, but the
   import removal makes it visually look like URL-encoding has
   been removed entirely from this file. A brief comment near
   `download_gcs_object` ("URL-encoding now happens inside
   `download_gcs_object` via `encode_gcs_object_name_for_url`,
   moved to the shared helper") would prevent a future "wait, did
   we lose the encode?" worry.

## Verdict rationale

Right architectural shape (centralized primitives module with
sanitize-vs-validate two-tier separation, upload constructors that
guarantee the post-upload file_id round-trips through the validator,
prefix-membership as the access-control test surface), right
load-bearing fix at the most-trusted call sites, right dual-gate on
both the kwargs and metadata callback-params loops, broad test
coverage including the metadata-bypass-attempt regression. Wants
255-char-truncation comment (so a future widening doesn't lose the
filename-safety bound), idempotent-encoder behavior-change note for
operators with `%XX`-named files, debug-only log line for
operator-side validator-rejection diagnostics, `@`-rejection in the
bucket-name parser, `frozenset` for the blocked-params set, split
into two PRs for VERIA tracking ergonomics, and a one-line "encoding
moved" comment.

`merge-after-nits`
