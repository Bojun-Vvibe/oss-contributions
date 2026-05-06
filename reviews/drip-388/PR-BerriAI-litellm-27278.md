# BerriAI/litellm#27278 — Fix Gemini MIME detection for extensionless GCS URIs

- PR: https://github.com/BerriAI/litellm/pull/27278
- Head SHA: `7908bb18fc275298bdc2662f719e4a5745c302e6`
- Size: +411/-23
- Verdict: **merge-after-nits** (man)

## Shape

Closes the silent-failure gap where a Gemini/Vertex request supplying a `gs://`
URI without a recognizable file extension (e.g. an upload pipeline that names
objects by SHA256 with no suffix) ended up forwarded with no `mime_type` set,
causing Gemini to either refuse the request or misclassify the binary.

Three new helpers added to
`litellm/llms/vertex_ai/gemini/transformation.py`:

- `_parse_gs_uri(gs_uri)` at `:184-191` — strict bucket+object split, raises
  `ValueError` on malformed input (including missing object component).
- `_is_valid_gcs_bucket_name(bucket)` at `:194-211` — full GCS bucket-name
  validator (length 3–63 normally, 222 if dotted; lowercase-only; no IP-style
  numeric-only; no `..`). Defensive sanity-check before any network call so
  malformed user input doesn't trigger a stray GCS metadata lookup with attacker-
  controlled host portion.
- `_get_gcs_object_content_type(image_url, vertex_project, vertex_credentials)`
  at `:215+` — issues a metadata-only GCS HEAD/GET via the new module-level
  `_GCS_METADATA_VERTEX_BASE = VertexBase()` (cached at module import) to
  resolve content-type live when extension fallback fails.

Plumbing change to the chat transform: `_transform_messages` at
`litellm/llms/gemini/chat/transformation.py:103-110` now takes an optional
`litellm_params: Optional[dict]` so the GCS-resolution helper can read
`vertex_project`/`vertex_credentials` without going through global state.

## Notable observations

- The MIME-alias map `_GEMINI_MIME_TYPE_ALIASES = {"image/jpg": "image/jpeg"}`
  at `:65-67` is correct (Gemini rejects `image/jpg`) but should be a frozen
  set of (alias, canonical) pairs in case future cases like `image/x-png` →
  `image/png` come up. Today's one-entry dict signals "this map will grow".
- `_GCS_METADATA_VERTEX_BASE = VertexBase()` at `:64` is module-global —
  thread-safe iff `VertexBase` itself is. If `VertexBase` caches access tokens
  in a non-thread-safe container, two concurrent gemini-chat requests with
  different `vertex_project` IDs could race. Verify `VertexBase` uses an
  internal lock for token refresh.
- The bucket validator at `:194-211` correctly rejects `192.0.2.1`-style names
  (`re.fullmatch(r"\d+\.\d+\.\d+\.\d+", bucket)`) and double-dot
  (`if ".." in bucket: return False`) — both are real GCS denial cases. The
  upper-bound length check (`222 if "." in bucket else 63`) matches GCS spec.
- `from urllib.parse import quote` is imported at `:9` but I can't see its use
  in the visible 250 lines — presumably used downstream in
  `_get_gcs_object_content_type` for object-key URL encoding. Confirm it's
  actually used and not import-leftover.

## Concerns / nits

- The added `litellm_params` parameter on `_transform_messages` is optional
  (default `None`), which is correct for compat, but every existing caller
  threading messages through must be updated; the diff truncates before I can
  see all `_gemini_convert_messages_with_history` callers — confirm there
  isn't a code path that constructs messages outside this new code that would
  silently get `None` and skip GCS metadata resolution.
- Network call to GCS metadata adds latency to every Gemini request that
  references a `gs://` URI. The PR should add a small in-memory LRU on the
  `(bucket, object)` → `content_type` pair so repeated calls in a hot loop
  don't re-fetch.
- No test in the visible diff for the bucket-validator (the `192.0.2.1` and
  `..` rejection cases especially). These are short, pure functions — should
  be table-driven pytest with both accept and reject rows.
