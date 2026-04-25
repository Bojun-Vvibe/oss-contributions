# BerriAI/litellm #26493 — Extend caller-permission checks to service-account + tighten raw-body acceptance

- **Repo**: BerriAI/litellm
- **PR**: [#26493](https://github.com/BerriAI/litellm/pull/26493)
- **Head SHA**: `bd914a6fab8f654657475aa4e95a19674e835a70`
- **Author**: yuneng-berri
- **State**: OPEN (+33 / -6)
- **Verdict**: `merge-as-is`

## Context

Follow-up to #26492 (already merged), which closed the
`key_type='management'` privilege-escalation hole on `/key/generate`
by adding a post-`handle_key_type` re-check. Reviewers spotted three
remaining gaps in the same area:

1. `_check_allowed_routes_caller_permission` permanently allowed the
   "safe presets" (`llm_api_routes`, `info_routes`) — but those
   presets are produced by `handle_key_type` *internally*, so
   accepting them at the *raw body* call site means a non-admin
   could write `allowed_routes=["llm_api_routes"]` directly and slip
   past the gate without going through `key_type` at all.
2. `/key/service-account/generate` (`generate_service_account_key_fn`)
   never called the caller-permission helpers, so a team-admin
   could create a service-account key with
   `metadata.allowed_passthrough_routes` populated and reach
   admin-only routes.
3. `/key/regenerate` had no post-`handle_key_type` re-check, so even
   though the column doesn't exist today, a future schema change
   would re-open the management-bucket escalation.

## Design

The core insight is the new `allow_safe_presets: bool` keyword-only
parameter on `_check_allowed_routes_caller_permission`
(`litellm/proxy/management_endpoints/key_management_endpoints.py:462-490`):

```python
def _check_allowed_routes_caller_permission(
    allowed_routes: Optional[list],
    user_api_key_dict: UserAPIKeyAuth,
    *,
    allow_safe_presets: bool = False,
) -> None:
```

Default `False` flips the previous default. Now:

- Raw-body call sites (`/key/generate` entry,
  `_validate_update_key_data`, `regenerate_key_fn`, and the new
  `/key/service-account/generate` entry call) get strict mode —
  non-admins can't write `allowed_routes` of any value directly.
- The single post-`handle_key_type` site inside
  `_common_key_generation_helper` (line 727) opts in with
  `allow_safe_presets=True`, which is the only path that can have
  legitimately *derived* `llm_api_routes` / `info_routes` from a
  non-admin's `key_type='llm_api'` or `key_type='read_only'` request.

Service-account fix at lines 1512-1520: lifts the same
`_check_allowed_routes_caller_permission` +
`_check_passthrough_routes_caller_permission` pair that
`/key/generate` already runs, no preset opt-in (because the helper
runs before any `handle_key_type` call). Symmetric with the parent
endpoint — that's the right shape.

Regenerate fix at lines 3970-3979: `handle_key_type(data, {}).get("allowed_routes")`
is invoked just to derive what the post-handle_key_type bucket
*would* be, then routed through the helper with
`allow_safe_presets=True`. The PR description explicitly notes this
is functionally a no-op today (prisma column doesn't exist) and
positioned as defense-in-depth. Right call — closing the surface
now is cheaper than tracking it later.

## Risks

- **Backward-compat for legitimate non-admin flows.** PR description
  enumerates: `key_type=llm_api` and `key_type=read_only` for
  non-admins still work because they go through the post-handler
  call site with `allow_safe_presets=True`. The 315-test pass
  confirms the parametrized matrix.
- **Default-change surprise.** Flipping the default from "permissive"
  to "strict" on a public-ish helper is the right move but anyone
  importing `_check_allowed_routes_caller_permission` from outside
  this module would now silently get stricter behavior. The
  underscore prefix signals "internal", and a grep of the PR shows
  every call site is updated, so this is fine.
- **Test e2e matrix.** PR description includes a manual e2e table
  showing 403 on the previously-permissive paths and 200 on the
  legitimate `key_type=llm_api` non-admin path. That's the right
  cross-product to verify.

## Suggestions

None blocking. Tiny nit: the docstring change (lines 470-489) is
clear about *why* the safe-presets carve-out exists and where it's
applied, which is exactly the documentation a future reader needs.
Worth keeping that level of detail.

## What I learned

Default-flip on a security helper is one of the safer ways to
close a "the gate accepts a value the caller could never
legitimately write directly" bug — far better than chasing every
call site individually with `# WARNING: don't pass user input
here`. The keyword-only `*, allow_safe_presets=False` shape forces
new callers to consciously opt in, which is the correct ergonomic
for a hardening primitive.
