# Review: BerriAI/litellm #27084 — Stop accepting user_config in client request bodies

- **Repo**: BerriAI/litellm
- **PR**: #27084
- **Head SHA**: `859fff159be5d69010814a5d6b9790aa12a8145f`
- **Author**: stuxf

## What it does

Removes the long-standing `user_config` request-body parameter that let a
caller hand the proxy a full `litellm.Router` config (model_list,
api_base, fallbacks, …). The proxy would instantiate a transient router
per request from that config, completely bypassing the central
`llm_router`'s RBAC, budget, and model-access enforcement. This PR rips
out that capability.

## Diff notes

- `litellm/proxy/auth/auth_utils.py:182` — drops `"user_config"` from
  `_BANNED_REQUEST_BODY_PARAMS`. That's because it no longer needs to be
  *banned with a 400* — it's now silently stripped at the parser
  boundary instead, which is a softer break for callers who were
  unknowingly sending it.
- `litellm/proxy/common_utils/http_parsing_utils.py:15-29` — defines
  `_ROOT_KEYS_TO_STRIP_FROM_REQUEST_BODY = ("user_config",)` and the
  `_strip_disallowed_root_keys()` helper, called from inside
  `_read_request_body()` after JSON parse and before caching. The
  inline comment is excellent — explains *why* (no internal code path
  ever sets `user_config`, it was always client-supplied → entire
  capability removed at the parser).
- `litellm/proxy/route_llm_request.py` — removes
  `_route_user_config_request()` and the `elif "user_config" in data`
  branch in `route_request()`. Correctly preserves the batch-completions
  branch (comma-separated models) which previously had a comment about
  running before the user_config check; that ordering becomes moot.
- `tests/proxy_unit_tests/test_proxy_pass_user_config.py` — entire test
  file deleted. Appropriate, since the capability is gone. **But**:
  there's no positive test added that asserts the new strip behavior
  (i.e., a request with `user_config` in the body succeeds and is
  routed through `llm_router`, ignoring the field). Should add one
  before merge.

## Concerns

1. **Backward compat**: Any caller currently relying on `user_config`
   will silently lose the override. The strip is silent (no log, no
   warning, no header). Suggest `verbose_proxy_logger.warning(...)`
   when stripping, gated on a one-shot per-process flag to avoid log
   floods.
2. **Documentation**: This is a security-relevant removal — needs a
   CHANGELOG note and a docs PR if `user_config` was ever documented.
3. **Test coverage**: Add a test that POSTs a body containing
   `user_config` and asserts (a) the request succeeds, (b) the response
   came from `llm_router`'s configured model, (c) no transient router
   was instantiated.

The change itself is correct and well-justified. Closing a router-injection
side channel that bypassed RBAC is a clear win.

## Verdict

merge-after-nits
