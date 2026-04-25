# BerriAI/litellm#26467 — Harden pass-through target URL construction

**Verdict: merge-as-is**

- PR: https://github.com/BerriAI/litellm/pull/26467
- Author: yuneng-berri
- Base SHA: `9521b74e9af60c479632459bf8358283b3885dd6`
- Head SHA: `5be4ed911e135a1ef050b820e394fda9d185d341`
- +110 / -37

## Summary

Closes two path-traversal-class bugs in the proxy's pass-through
endpoint URL construction: (1) `construct_target_url_with_subpath`
concatenated raw subpaths onto a configured `base_target` without
resolving `..`, and (2) every per-provider passthrough handler
(Anthropic, Gemini, Mistral, Cohere, Cursor, Milvus, AssemblyAI,
Bedrock, OpenAI factory) built the upstream URL via
`base_url.copy_with(path=encoded_endpoint)`, which **replaces** the
entire path component on the configured base — silently discarding any
operator-configured prefix. Adds
`HttpPassThroughEndpointHelpers.join_base_and_endpoint_path()` and
routes every handler through it; also normalizes the subpath in
`construct_target_url_with_subpath` with `posixpath.normpath` and
clamps back to `base_path` if normalization climbs above it.

## Specific findings

- `litellm/proxy/pass_through_endpoints/pass_through_endpoints.py:+621/-602`
  (head SHA `5be4ed911e135a1ef050b820e394fda9d185d341`) — the new helper:
  ```
  combined = posixpath.normpath(base_path + "/" + clean_endpoint)
  if combined != base_path and not combined.startswith(base_path + "/"):
      return base_path + "/"
  ```
  is the load-bearing line. The `base_path + "/"` containment check
  prevents the classic `posixpath.normpath` bypass where `/foobar`
  passes a naive `startswith("/foo")` test (because there's no
  separator boundary). Correct.
- Same file, `construct_target_url_with_subpath:+603/-1` —
  ```
  trailing_slash = subpath.endswith("/")
  safe_subpath = posixpath.normpath("/" + subpath).lstrip("/")
  if safe_subpath == ".":
      safe_subpath = ""
  ```
  The `posixpath.normpath("/" + subpath)` rooting trick is the right
  way to ensure `..` segments at the head of `subpath` resolve back to
  `/` rather than escaping. Trailing-slash preservation is correct
  (some upstream APIs distinguish `/v1/foo` from `/v1/foo/`).
- `litellm/proxy/pass_through_endpoints/llm_passthrough_endpoints.py:+~/-~` —
  every provider arm goes from `updated_url = base_url.copy_with(path=encoded_endpoint)`
  to `updated_url = base_url.copy_with(path=HttpPassThroughEndpointHelpers.join_base_and_endpoint_path(base_url, encoded_endpoint))`.
  The Anthropic, Gemini, Mistral, Cohere, Cursor, Milvus, AssemblyAI,
  and Bedrock arms are all updated. The `_join_url_paths` helper for
  the OpenAI flow is also routed through the new central helper, so
  the OpenAI-specific suffix logic (`responses` / `chat/completions`)
  retains its trailing path handling. I see no provider arm left
  behind — though a maintainer should grep for any remaining
  `copy_with(path=encoded_endpoint)` to be sure.
- Test verification: 227 passed in
  `tests/test_litellm/proxy/pass_through_endpoints/`. Worth noting the
  PR description doesn't list a *new* test for traversal payloads —
  the manual exercise in the PR body is good but should be encoded as
  pytest cases (`../../etc/secret`, `foo/../../../etc/secret`,
  `////foo`, `\u002e\u002e/etc/secret` URL-encoded variant). Not a
  blocker because the existing 227 tests cover the URL-construction
  surface, but adding 3-4 explicit traversal cases would lock the
  fix in.

## Rationale

Two real bugs, one fix, in the right shape. The base-path-discard bug
is the more impactful of the two: any operator who configured
`base_target_url: "https://upstream.example.com/v2/api"` was silently
seeing requests routed to `https://upstream.example.com/<endpoint>`
because httpx's `copy_with(path=...)` replaces the path component
wholesale. That's been a footgun for as long as the per-provider
handlers existed, and centralizing on a single helper closes the
class. The traversal hardening on `construct_target_url_with_subpath`
is correct in shape — `posixpath.normpath` after rooting with `/`,
then containment-check against `base_path + "/"` to avoid the
prefix-as-substring trap. Trailing-slash preservation is the right
call because some upstream APIs (notably AssemblyAI and parts of
Bedrock's invoke namespace) treat `foo` and `foo/` as distinct
resources. The 227-test passing run gives reasonable confidence and
the helper is small enough to audit by eye. Three followup items I'd
file as separate issues, not blockers: (a) add explicit traversal-
payload pytest cases so a future regression on the helper is caught
locally, (b) emit a `verbose_logger.warning` when the
`return base_path + "/"` clamp triggers — silent clamp on a
suspicious request is the same shape as silently dropping headers,
and (c) consider a one-line `verbose_logger.debug` of the
pre-resolution vs post-resolution path so operators can debug
mis-configured `base_target_url` values. Otherwise this is a clean
two-bug fix that should land.
