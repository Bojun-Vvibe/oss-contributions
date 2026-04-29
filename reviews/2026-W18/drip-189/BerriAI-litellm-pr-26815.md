# BerriAI/litellm#26815 — chore(proxy): contain UI_LOGO_PATH / LITELLM_FAVICON_URL on unauthenticated asset endpoints

- PR: https://github.com/BerriAI/litellm/pull/26815
- HEAD: `14473ed8f964e1e345f90a7a190680fadf86b45b`
- Author: stuxf
- Files changed: 6 (+515 / -71) — new `litellm/proxy/common_utils/static_asset_utils.py` (+137),
  `proxy_server.py` (+80/-67), `config_override_endpoints.py` (+9/-2), tests (+289).

## Summary

Closes three CVE-class bugs on the unauthenticated logo/favicon endpoints
(GHSA-3pcp-536p-ghjc LFI, GHSA-pjc9-2hw6-78rr SSRF):

1. **`/get_image` and `/get_favicon`** — local file paths via `UI_LOGO_PATH`
   were gated only on `os.path.exists(env_var)`, so an admin (or hostile
   env-var injection) could set `UI_LOGO_PATH=/etc/passwd` and any
   unauthenticated caller could exfiltrate it. Now goes through
   `resolve_local_asset_path` with realpath-based containment in
   `[assets_dir, current_dir]`.
2. **HTTP fetches from the same env vars** — bytes returned verbatim with
   hardcoded `image/jpeg` Content-Type. `UI_LOGO_PATH=http://internal-svc/...`
   exfiltrated whatever that returned. Now goes through
   `fetch_validated_image_bytes` which delegates to `async_safe_get`
   (per-redirect-hop SSRF revalidation against `BLOCKED_NETWORKS`) and
   enforces an `ALLOWED_IMAGE_CONTENT_TYPES` allowlist.
3. **`/get_logo_url`** — was returning the raw `UI_LOGO_PATH` value,
   leaking internal hostnames or filesystem paths. Now only returns
   HTTP(S) URLs; local paths return `""` and the dashboard falls back to
   `/get_image` (which serves the contained file).

The Vault test endpoint also gets `async_safe_get` retrofit at
`config_override_endpoints.py:393-406`.

## Cited hunks

- `litellm/proxy/common_utils/static_asset_utils.py:32-49` — the
  `ALLOWED_IMAGE_CONTENT_TYPES` frozenset with a load-bearing comment
  block at `:38-46` explaining why `image/svg+xml` is intentionally
  excluded ("SVG is the only common image format that can embed
  JavaScript, and the endpoint is unauthenticated"). Excellent — this is
  the kind of design choice that gets silently flipped in a future
  "support more image types" PR without that comment present.
- `litellm/proxy/common_utils/static_asset_utils.py:51-81` —
  `resolve_local_asset_path` uses `os.path.realpath` (correctly follows
  symlinks before checking containment) and the prefix check uses
  `resolved.startswith(root_resolved + os.sep)` (correctly avoids the
  `/var/data-evil` matches `/var/data` foot-gun). Two small weaknesses:
  the `os.path.expanduser` at `:62` runs as the proxy process user and
  resolves `~` differently than admins might intend (probably fine since
  this is admin-set config), and the `OSError, ValueError` swallow on
  `realpath` doesn't differentiate "path is malformed" from "path
  traverses a permission boundary" — the former should fall back, the
  latter probably should warn-log too.
- `litellm/proxy/common_utils/static_asset_utils.py:96-117` —
  `fetch_validated_image_bytes` honors `litellm.user_url_validation` and
  defaults `timeout_s=5.0`. The defensive `if not isinstance(raw_content_type, str)`
  at `:128-132` correctly handles upstream Mock-shaped responses too.
- `litellm/proxy/proxy_server.py:12224-12239` — `/get_logo_url` redacts
  to HTTP(S)-only and returns `""` for local paths. The 11-line docstring
  at `:12224-12233` names exactly why ("the dashboard falls back to
  `/get_image` which serves the file with path containment instead").
  Same redaction-with-rationale pattern as the asset-utils SVG comment.
- `litellm/proxy/proxy_server.py:12283-12330` — `/get_image` now imports
  the helpers locally (avoiding a top-level circular import — common
  pattern in this file but worth flagging that lazy imports inside hot
  endpoints add per-request overhead vs a top-level import) and the
  fallback `verbose_proxy_logger.warning` at `:12302-12305` correctly
  uses `%r` formatting (`logo_path`) so a path containing format
  specifiers can't break the log line.
- `litellm/proxy/management_endpoints/config_override_endpoints.py:393-410`
  — Vault `lookup-self` retrofit to `async_safe_get`. The comment at
  `:393-398` correctly names the failure mode (admin-set `vault_addr`
  pivoting to cloud metadata) and the operator escape hatch
  (`litellm.user_url_allowed_hosts`).

## Risks

- The `default_logo` short-circuit at `proxy_server.py:12286` (line numbers
  approximate from the diff context) is the path that bypasses
  containment. Confirm `default_logo` is computed from a constant, not
  from any env var that could be hijacked — otherwise the LFI window
  reopens via the default path.
- `ALLOWED_IMAGE_CONTENT_TYPES` includes `image/jpg` (non-IANA, but
  emitted by some CDNs). Fine, but worth a follow-up to log-and-warn
  when the upstream emits non-canonical types so operators can see drift.
- The local imports inside `/get_image` (the `from litellm.proxy.common_utils.static_asset_utils import ...`)
  add ~tens of microseconds per request. For an unauthenticated
  high-traffic logo endpoint this is unlikely to matter but a top-level
  import would be cleaner if the circular-import concern can be resolved.
- 192 + 215 lines of new tests is generous coverage but no integration
  test that actually starts the proxy and curls `/get_image?path=/etc/passwd`
  — the unit tests cover the helpers in isolation. A single end-to-end
  smoke test would fence against future refactors that bypass the
  helpers.

## Verdict

**merge-after-nits**

## Recommendation

Land after (1) confirming `default_logo` is constant-derived not
env-var-derived, (2) adding one e2e smoke test asserting
`GET /get_image` with `UI_LOGO_PATH=/etc/passwd` returns the default not
the file, and (3) optionally moving the local imports to top-level if
circular imports allow. The design — allowlist-not-denylist for both
content types and asset roots, realpath-based containment, and
`async_safe_get` delegation — is the right shape, and the colocated
SVG-XSS rationale comment is the kind of thing future maintainers will
thank this PR for.
