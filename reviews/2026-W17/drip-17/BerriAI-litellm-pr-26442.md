# BerriAI/litellm PR #26442 — feat: restrict org admins from creating keys, teams, models via UI settings

- **Author:** ryan-crabbe-berri
- **Head SHA:** f9bce31a7927ae7ba05d1992ee2206e064af6adb
- **Files:** `litellm/proxy/common_utils/rbac_utils.py` (+41),
  `litellm/proxy/management_endpoints/key_management_endpoints.py` (+5),
  `litellm/proxy/management_endpoints/model_management_endpoints.py` (+5),
  `litellm/proxy/management_endpoints/team_endpoints.py` (+5),
  `litellm/proxy/ui_crud_endpoints/proxy_setting_endpoints.py` (+21),
  `tests/litellm/proxy/common_utils/test_rbac_utils.py` (+85 / −2),
  `tests/test_litellm/proxy/ui_crud_endpoints/test_proxy_setting_endpoints.py` (+54)
- **Verdict:** `merge-after-nits`

## What the diff does

Introduces a per-feature opt-in lockout for the `ORG_ADMIN` role,
toggleable from the UI settings page.

In `rbac_utils.py` lines 14–21 a new `OrgAdminFeatureName` literal
union (`key_generate | team_create | model_add`) and a label map
are added. The new helper `check_org_admin_feature_access`
(lines 79–110):

```python
if user_api_key_dict.user_role not in (
    LitellmUserRoles.ORG_ADMIN,
    LitellmUserRoles.ORG_ADMIN.value,
):
    return
disable_flag = f"disable_{feature_name}_for_org_admin"
if not general_settings.get(disable_flag, False):
    return
raise HTTPException(status_code=403, ...)
```

It's then called once per endpoint:
- `key_management_endpoints.py:1260` inside `generate_key_fn` after
  the `verbose_proxy_logger.debug("entered /key/generate")` line
  and before the budget validation.
- `model_management_endpoints.py:978` inside `add_new_model` after
  the early auth-shape check and before
  `ModelManagementAuthChecks.can_user_make_model_call`.
- `team_endpoints.py:900` inside `new_team` right after the
  `prisma_client is None` guard.

`proxy_setting_endpoints.py` (+21) wires the three new
`disable_*_for_org_admin` flags into the proxy settings GET/PATCH
surface so the UI toggle persists.

Tests cover both the `rbac_utils` helper (85 lines: role gate,
flag-off short-circuit, flag-on rejection) and the proxy settings
endpoint round-trip (54 lines).

## Review notes

- **Role gate is the right safety choice** at lines 95–99. Checking
  both `LitellmUserRoles.ORG_ADMIN` (enum) and
  `LitellmUserRoles.ORG_ADMIN.value` (string) handles both code
  paths reliably; this kind of dual-check is exactly what keeps
  enum/string drift from creating silent bypasses.
- **Feature short-circuit defaults to allow** (`general_settings.get
  (disable_flag, False)` line 105). Correct default — opt-in
  restriction, no behavior change for existing deployments.
- **Auth check ordering matters.** In `model_management_endpoints.py`
  the new check runs *before* `can_user_make_model_call`, which
  means an org admin with the lockout enabled gets a clean
  "model creation is disabled for org admins" 403 instead of a
  potentially misleading model-policy error. Good ordering. Same
  comment for `team_endpoints.py:900` — runs after only the
  prisma-client null guard, so the rejection isn't masked by other
  preconditions.
- In `key_management_endpoints.py`, the call sits *before* the
  `data.max_budget < 0` validation. That means an org admin with
  the flag on hits 403 before any input-shape complaint, which is
  the right user experience.
- **Error message hardcodes "Contact your proxy admin."** — fine,
  but if the deployment has a custom escalation channel, there's
  no way to inject that. Consider a `general_settings.get
  ("org_admin_lockout_message")` override, but not blocking.
- **Three flags vs one set.** The disable flag pattern
  `disable_{feature}_for_org_admin` works, but a single
  `org_admin_disabled_features: list[str]` would be cleaner config
  and easier to extend. Existing `_ORG_ADMIN_FEATURE_LABELS` already
  centralizes the names, so refactoring would be cheap. Worth
  considering before this surface area grows.
- **Tests look thorough.** 85 lines on `test_rbac_utils.py` adding
  `test_check_org_admin_feature_access_*` cases plus the settings
  round-trip is the right ratio for a security-adjacent change.
  Confirm there's a test for "user_role is None" — mid-stack RBAC
  bugs often slip in there.
- **Naming nit.** `OrgAdminFeatureName` is fine but the constant
  `_ORG_ADMIN_FEATURE_LABELS` with a leading underscore implies
  module-private — which it is, but it's also the only humanized
  name source for the 403 detail body. If anything else (audit
  log, UI surface) ever needs the same labels, promote it to
  public.

Solid scoped RBAC addition. Ship after addressing the
single-flag-list refactor question (or a comment explaining the
per-flag choice).

## What I learned

Adding a role-scoped feature gate to multiple endpoints is one of
those changes where call-site ordering is half the design — the
gate must run after the role is known and before the expensive /
side-effect-bearing work, but ideally before validation errors
that would mask the real reason for rejection. This PR gets that
ordering right at all three sites.
