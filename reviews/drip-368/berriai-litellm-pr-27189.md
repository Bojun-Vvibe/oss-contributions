# BerriAI/litellm PR #27189 — fix: enforce non-admin RBAC in `model_info_v2`

- Head SHA: `9a9323022f5096c467cabbe0343b8e0129688075`
- Author: `orbisai0security`
- Size: +4 / -0
- Verdict: **merge-after-nits**

## Summary

Fixes a real RBAC bypass in the proxy's `/model/info/v2` (key
management / model-info) endpoint. Before this change, a non-admin
user could call the endpoint with `user_models_only=false` (the
default) and receive the full proxy model catalog including model
entries they had no team/key permission to access — a clean
information-disclosure: model names, provider routing, in some
deployments raw `litellm_params` keys (rate limits, fallbacks,
upstream endpoints, sometimes embedded credentials in dev configs).

## What the diff actually does

In `litellm/proxy/proxy_server.py:11008-11014` the PR inserts a
3-line guard immediately before the existing
`if user_models_only:` filtering branch:

```python
# Enforce RBAC: non-admin users should only see their accessible models
if user_api_key_dict.user_role != LitellmUserRoles.PROXY_ADMIN:
    user_models_only = True
```

The role check is the right gate: `PROXY_ADMIN` is the canonical
"sees everything" role across litellm's `LitellmUserRoles` enum
(team admins / org admins / internal users / proxy users all
fall through to the `True` branch and get filtered to their
accessible-models subset via the existing
`non_admin_all_models(...)` helper at line 11017). The PR doesn't
touch `non_admin_all_models` itself, so the existing
team/key/budget filtering semantics are preserved verbatim — this
is purely a "default the flag for non-admins" enforcement, which
is the minimal-blast-radius shape for a security fix.

## Why merge-after-nits (not merge-as-is)

The fix is correct in substance but the patch as posted has gaps
that should land before this is shipped:

1. **No regression test.** A 4-line security fix in the proxy
   surface should land with at least one pytest case in
   `tests/proxy_unit_tests/` that hits `model_info_v2` with a
   `LitellmUserRoles.INTERNAL_USER` (or any non-admin) key and
   `user_models_only=False` and asserts the returned model list is
   filtered. Without this, the next refactor of the
   `user_models_only` path can quietly re-open the hole — and the
   changelog will say "no behavior change" because there's no
   golden test.

2. **No mention of `PROXY_ADMIN_VIEW_ONLY`.** litellm has a
   `PROXY_ADMIN_VIEW_ONLY` role distinct from `PROXY_ADMIN`. Read
   literally, the new check forces `user_models_only=True` for
   that role too, which **may be intentional** (view-only admin =
   only sees their accessible models) but **may be wrong** (the
   role's purpose is "read-only full visibility"). Author should
   confirm and either use `not in (PROXY_ADMIN, PROXY_ADMIN_VIEW_ONLY)`
   or document that view-only admins are deliberately filtered
   alongside other non-admins.

3. **PR title is truncated** (`fix: the proxy server exposes key
   management and mod... in...`) — should be expanded before merge
   to something concrete, e.g. `fix(proxy): enforce non-admin RBAC
   on /model/info/v2 to prevent model-catalog disclosure`. Helps
   downstream CVE / changelog triage.

4. **No security-advisory reference.** If this was reported as a
   security issue (the author handle `orbisai0security` strongly
   suggests so), the PR description should at minimum link the
   GHSA / CVE id even if the maintainers will assign one on merge,
   so the fix is discoverable in `git log --grep`.

## Optional nits (not blocking)

- The comment `# Enforce RBAC: non-admin users should only see their
  accessible models` could be tighter: `# RBAC: non-admin callers
  cannot opt out of user_models_only filtering` makes the
  *enforcement* angle (you can no longer pass
  `user_models_only=False` as non-admin to bypass) explicit.
- Worth auditing sibling endpoints (`/model/info`,
  `/v1/model/info`, `/model_group/info`) for the same
  `user_models_only`-defaults-to-False-and-trusted shape, in a
  follow-up PR. The same class of bug is likely to recur there.
