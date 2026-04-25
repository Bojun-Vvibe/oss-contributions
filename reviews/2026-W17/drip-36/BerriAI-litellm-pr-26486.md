# BerriAI/litellm #26486 — admin-only gate on key_type, allowed_passthrough_routes, and key/regenerate grant fields

- **Repo**: BerriAI/litellm
- **PR**: [#26486](https://github.com/BerriAI/litellm/pull/26486)
- **Head SHA**: `6fde9395f18c7fb8d534da7299549f4b551243f9`
- **Author**: stuxf
- **State**: OPEN (+460 / -55)
- **Verdict**: `merge-after-nits`

## Context

Three independent privilege-escalation primitives in the key-mint /
key-regenerate / pass-through-route paths, all closable by adding
admin-role gates on specific request fields:

1. `POST /key/generate` accepts `key_type='management'`, which
   `handle_key_type` later expands into
   `allowed_routes=['management_routes']`. The existing
   `_check_allowed_routes_caller_permission` inspects the *raw*
   `allowed_routes` request field — by the time `key_type` is
   expanded, the gate has already passed. So a non-admin can request
   `key_type='management'` with an empty `allowed_routes` and end up
   minting a management-scope key.

2. `metadata.allowed_passthrough_routes` is consumed by
   `RouteChecks.check_passthrough_route_access` *before* the
   standard role-based route gate. A non-admin who can set this
   metadata can list any internal endpoint and authorize themselves
   against it.

3. The pass-through-route check itself doesn't verify that the
   listed route is actually a registered pass-through endpoint —
   admin internal routes like `/cache/settings` or `/model/new` can
   slip through.

## Design

In `litellm/proxy/auth/route_checks.py:537-575` (lines 9-32 of the
diff) the pass-through-route check gains a registry-membership
gate:

```py
from litellm.proxy.pass_through_endpoints.pass_through_endpoints import (
    InitPassThroughEndpointHelpers,
)
...
if not InitPassThroughEndpointHelpers.is_registered_pass_through_route(route):
    return False
```

This is the right spot: even with the metadata-gate fix in (2),
this is the defense-in-depth that prevents a stale or
admin-mutated metadata blob from authorizing arbitrary routes.
The lazy import inside the function is a circular-dep dodge —
common in this codebase but worth noting.

In
`litellm/proxy/management_endpoints/key_management_endpoints.py:484-549`
two new gates:

- `_check_management_key_type_caller_permission` (lines 45-71)
  rejects non-admins requesting `key_type=LiteLLMKeyType.MANAGEMENT`
  with a 403 and a *useful* error message that points at the
  alternatives (`'llm_api'`, `'read_only'`). Lines 59-62 are the
  fast path: not management → return; admin → return; otherwise
  raise. Order is correct.
- `_check_allowed_passthrough_routes_caller_permission` (lines
  74-102) does the same for the metadata field. Lines 88-91 short-
  circuit on falsy metadata or empty list, so the common case (no
  pass-through routes requested) is free.

The docstrings are unusually good — they explain not just what the
gate does but *why the existing checks didn't catch it* (timing
relative to `handle_key_type`, ordering relative to the standard
role-route gate). That context matters for future maintainers
reasoning about whether to remove or relax the gates.

## Risks

- **Call-site coverage**: I only see the helpers defined here; need
  to confirm both `_check_management_key_type_caller_permission`
  and `_check_allowed_passthrough_routes_caller_permission` are
  called from *every* code path that mints or regenerates a key —
  `/key/generate`, `/key/regenerate`, `/key/update`, and any
  internal helper that constructs a key without going through the
  endpoint (e.g. team key auto-mint). The PR title mentions
  `key/regenerate grant fields` so that path is in scope; would be
  useful to grep for callers in the test additions to confirm both
  endpoints invoke the new gates.
- **Lazy-import cost**: importing
  `InitPassThroughEndpointHelpers` inside
  `check_passthrough_route_access` is fine for the cold path, but
  this function runs per-request on the auth hot path. Worth either
  caching the import at module-bottom (after the circular import
  resolves) or memoizing the registry lookup.
- **Error-message disclosure**: the 403 detail at lines 67-69 names
  `'management'` as the privileged value. That's information a
  non-admin probably already has from public docs, but minimizing
  detail is the standard auth-error convention.
- **Test coverage**: 460 added lines including
  `test_route_checks.py` and `test_key_management_endpoints.py` —
  good signal, would want to spot-check that both the
  registry-membership-fail and the role-fail branches each have a
  test.

## Verdict

`merge-after-nits` — security-positive, well-explained, the only
items are call-site-coverage confirmation, the lazy-import nit, and
an error-message-detail review.
