# BerriAI/litellm PR #27007 — fix(auth): block missing write routes for proxy admin viewers

- **Head SHA:** `2a8fe32850a7fef65f305bfaa08923dd6e513d97`
- **Files:** 3 (`_types.py`, `auth/route_checks.py`, `tests/.../test_route_checks.py`)
- **LOC:** +137 / −0

## Observations

- `route_checks.py:18-50` — the new `_PROXY_ADMIN_VIEW_ONLY_BLOCKED_ROUTES` frozenset is a real fix for a real bug: the previous structure of `_check_proxy_admin_viewer_access` had a fall-through that allowed view-only admins to call `/team/block`, `/team/unblock`, `/team/permissions_update`, `/team/permissions_bulk_update`, `/key/bulk_update`, the `/jwt/key/mapping/*` write routes, and the templated `/key/{id}/regenerate` and `/key/{id}/reset_spend`. That is a significant privilege-escalation gap for any deployment using `PROXY_ADMIN_VIEW_ONLY`.
- `route_checks.py:711-734` — the gating logic checks `RouteChecks.check_route_access(route, LiteLLMRoutes.management_routes.value)` first, then either allows `/user/update` with whitelist of params (`user_email`, `password`), denies if matched in the new blocklist, or falls through to allow. The fall-through is now narrower (read routes only) — correct.
- `route_checks.py:46-50` — `_PROXY_ADMIN_VIEW_ONLY_BLOCKED_KEY_SUFFIXES = ("/regenerate", "/reset_spend")` plus the `route.startswith("/key/") and route.endswith(...)` check is the right way to handle path-templated routes that the enum stores as `{key_id}` literals. The test at `test_route_checks.py:117-118` exercises both `/key/abc123/regenerate` and `/key/abc123/reset_spend`.
- `route_checks.py:21-23` — the comment `"Adding a new write endpoint to a management router REQUIRES adding it here too"` is good defensive guidance, but this remains a **manual sync invariant** that will silently break the next time someone adds a new write route. A follow-up improvement worth filing: introduce a `is_write` flag on the route enum so the blocklist becomes derived rather than hand-curated. Non-blocking for this PR.
- `_types.py:568` — appending `/team/permissions_bulk_update` to `LiteLLMRoutes.management_routes` is necessary so the gate above even runs for that route. Without this line, the blocklist entry would be unreachable. Good defensive coupling.
- `test_route_checks.py:93-159` — parametrized tests cover 14 blocked routes and 12 allowed read routes. Strong.

## Verdict: `merge-as-is`

- Closes a real privilege-escalation gap with proportionate scope and proportionate tests. The hand-curated blocklist is a known weakness but is documented; a follow-up to derive it from route metadata would be a cleaner long-term fix.
