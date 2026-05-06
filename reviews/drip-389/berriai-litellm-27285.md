# BerriAI/litellm#27285 — configurable env for always_include_stream_usage

- Head SHA: `bc5169384548d427229f70d79e5429f36d5ab4ce`
- Author: @joshhatfield
- Link: https://github.com/BerriAI/litellm/pull/27285

## Notes
- `litellm/constants.py:1684` reads `ROUTER_ALWAYS_INCLUDE_STREAM_USAGE` at import time via `os.getenv(...).lower() == "true"`. Import-time env reads are brittle for tests and for runtime config reloads — the test at `tests/test_litellm/proxy/test_common_request_processing.py:909` even has to `monkeypatch.setattr(constants, ...)` to work around it. Prefer reading the env var inside the call site or wrapping in a `@lru_cache` getter.
- `litellm/proxy/common_request_processing.py:920–921` adds `or ROUTER_ALWAYS_INCLUDE_STREAM_USAGE is True`. Using `is True` against a bool is fine but redundant; just `or ROUTER_ALWAYS_INCLUDE_STREAM_USAGE` reads clearer.
- Naming: `ROUTER_*` implies router-scoped, but it's actually applied in the proxy request path regardless of router. Either rename to `LITELLM_ALWAYS_INCLUDE_STREAM_USAGE` or scope to actual `Router` calls.

## Verdict
`request-changes`

Functional intent is good (env-driven default), but import-time evaluation breaks dynamic config reload and forces the test to monkeypatch the module attribute. Move the read into a helper called per-request, and reconsider the `ROUTER_` prefix.
