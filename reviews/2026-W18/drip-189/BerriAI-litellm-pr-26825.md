# BerriAI/litellm#26825 — chore(auth): gate oauth2-proxy header trust on premium + privileged-field denylist

- PR: https://github.com/BerriAI/litellm/pull/26825
- HEAD: `fbcfd59b1a23edc17d6a93880726f26e308efedd`
- Author: stuxf
- Files changed: 2 (+261 / -16) — `litellm/proxy/auth/oauth2_proxy_hook.py` (+69/-16),
  `tests/test_litellm/proxy/auth/test_oauth2_proxy_hook.py` (+192 new)

## Summary

Closes a real privilege-escalation class on the oauth2-proxy header-trust
auth path (CVE label `GHSA-5c3m-qffq-4r9m` referenced in the test
docstring). Two failures:

1. **No `premium_user` gate**, unlike sibling auth paths
   (`enable_oauth2_auth`, `enable_jwt_auth`) — open-source deployments
   could enable header-trust without realising it requires a hardened
   reverse-proxy topology.
2. **No allowlist on which `UserAPIKeyAuth` fields can be mapped from
   request headers** — an admin who maps the wrong header to `user_role`
   lets any caller send `X-User-Role: proxy_admin` and have Pydantic
   coerce it into `LitellmUserRoles.PROXY_ADMIN`. Same shape for
   `max_budget`, `permissions`, `api_key`, etc.

The fix introduces a six-element identity-only `ALLOWED_OAUTH2_PROXY_FIELDS`
frozenset (`user_id`, `user_email`, `team_id`, `team_alias`, `org_id`,
`models`) and rejects mappings whose keys aren't in it at request time
(loud failure, not silent privesc). Adds 192 lines of regression tests.

## Cited hunks

- `litellm/proxy/auth/oauth2_proxy_hook.py:9-37` — the
  `ALLOWED_OAUTH2_PROXY_FIELDS` allowlist with a 19-line comment
  explaining (a) why it's an allowlist not a denylist (the auth model has
  ~50 budget/spend/limit/permission fields and grows; default-secure means
  new fields are blocked automatically), (b) what operators should do if
  they need richer assertions (switch to JWT auth which validates a
  signature). Excellent design rationale colocated with the constant.
- `litellm/proxy/auth/oauth2_proxy_hook.py:67-81` — the runtime check
  `disallowed = sorted(set(oauth2_config_mappings.keys()) - ALLOWED_OAUTH2_PROXY_FIELDS)`
  followed by `raise ValueError(...)` with a multi-line operator-friendly
  message naming the disallowed keys, the allowed set, and the JWT
  fallback. Crucially this raises at request time, not at config-load
  time — so a misconfiguration surfaces on the first auth attempt with
  the real attacker's headers in the log context, rather than silently at
  startup with no operator action.
- `litellm/proxy/auth/oauth2_proxy_hook.py:84-95` — the auth_data
  construction loop drops the previous `if key == "max_budget": float(value)`
  branch (since `max_budget` is no longer reachable through the
  allowlist). `models` retains its comma-split because it's the only
  identity field that's a list type.
- `tests/test_litellm/proxy/auth/test_oauth2_proxy_hook.py:1-21` — the
  test docstring spells out both failure modes including the explicit
  `LitellmUserRoles.PROXY_ADMIN` privesc path. Good for future
  reviewers who'll see this code without context.
- `tests/test_litellm/proxy/auth/test_oauth2_proxy_hook.py:42-60` — the
  `configure_proxy` fixture monkeypatches `general_settings` per-test
  rather than mutating module state, so test ordering can't introduce
  false greens.

## Risks

- The `premium_user` gate referenced in the PR body and test docstring
  doesn't appear in the visible diff slice. Need to confirm the gate is
  actually wired (e.g. a `if not premium_user: raise` check in the
  hook, or in the calling middleware that registers this hook). If it's
  missing from the implementation, the open-source-deployment failure
  mode (1) above stays open even after this lands.
- The `models` field stays in the allowlist but it *is* a privilege grant
  (it controls which models the caller can reach). Operators commonly use
  the upstream IdP's group claim → `models` mapping as a coarse
  authorization mechanism. That's a defensible identity-vs-privilege call
  to debate — `models` is identity-shaped (a list of strings the IdP
  asserts) but enforcement-shaped (it gates dispatch). A short note in
  the comment naming this tension would help.
- The error raised on disallowed mappings is a `ValueError`, which the
  upstream caller may render to the client. If `disallowed` includes
  attacker-controlled keys (it doesn't here — keys come from
  `oauth2_config_mappings` admin config, not from the request), this
  would be an injection vector. Currently safe but worth a
  `repr()`-wrapping in case future code paths route untrusted keys
  through the same constant.

## Verdict

**merge-after-nits**

## Recommendation

Land after (1) confirming the `premium_user` gate is actually wired (or
adding it if the PR body is ahead of the diff), (2) adding a one-line
comment at `models` in the allowlist naming the identity-vs-privilege
tension and the operator's responsibility to scope the upstream claim,
and (3) adding a regression test that explicitly asserts
`X-User-Role: proxy_admin` with a `user_role` mapping is rejected at
config-load (the headline scenario that motivates the PR). The allowlist
design is the right shape and the colocated rationale is exemplary.
