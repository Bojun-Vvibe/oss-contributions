# BerriAI/litellm #27275 — Allow internal users to view tag usage

- PR: https://github.com/BerriAI/litellm/pull/27275
- Head SHA: `c95f57f095b53e48762791cbc7baf90f3ad00b29`
- Base: `litellm_internal_staging`
- Size: +89 / -5 across 6 files
- Files:
  - `litellm/proxy/_types.py` (+6/-1)
  - `tests/test_litellm/proxy/auth/test_route_checks.py` (+37/-0)
  - `ui/litellm-dashboard/src/components/UsagePage/components/UsagePageView.tsx` (+8/-2)
  - `ui/litellm-dashboard/src/components/UsagePage/components/UsagePageView.test.tsx` (+21/-2)
  - `ui/litellm-dashboard/src/components/UsagePage/components/UsageViewSelect/UsageViewSelect.tsx` (+5/-0)
  - `ui/litellm-dashboard/src/components/UsagePage/components/UsageViewSelect/UsageViewSelect.test.tsx` (+12/-0)

## Verdict
**merge-after-nits**

## Rationale
The behavior change is well-scoped: the tag usage analytics view in the dashboard was admin-only by both UI gating and backend route allowlist, so internal users couldn't reach a feature that they have a legitimate need for. The PR fixes both layers consistently.

**Backend (`litellm/proxy/_types.py:672-687`):** Adds `/tag/daily/activity` and `/tag/list` to two allowlists — the broader `internal_user_allowed_routes` and the narrower `internal_user_view_only_routes`. Both endpoints are read-only (`/list`, `/daily/activity`), so widening view-only access is the right scope. The diff context makes clear these are appended alongside `/v2/guardrails/list`, `/project/list`, `/project/info` — same shape as existing read endpoints, no privilege drift.

**Frontend (`UsagePageView.tsx:84-101`, `UsageViewSelect.tsx:107-167`):** Introduces a new `canViewTagUsage = isAdmin || internalUserRoles.includes(userRole || "")` gate and threads a new optional `canViewTagUsage?: boolean` prop through `UsageViewSelect`. The filter at `UsageViewSelect.tsx:163-165` correctly special-cases `option.value === "tag"` before the generic `adminOnly` filter, so the existing admin-only behavior for *other* options is preserved.

**Tests:** Coverage is genuinely paired with the change:
- `test_route_checks.py:53-90` — parametrized over both `INTERNAL_USER` and `INTERNAL_USER_VIEW_ONLY`, both routes (`/tag/list`, `/tag/daily/activity`) — 4 cases total. Uses `pytest.fail` inside `try/except` which is a slightly verbose pattern (could just let the exception bubble) but unambiguous.
- `UsageViewSelect.test.tsx:113-138` — adds positive case (gate true → option visible) and negative case (gate false, non-admin → option hidden). Good.
- `UsagePageView.test.tsx` — adds an `it("should render tag usage option for internal users")` integration-style assertion at the page level.

The role-based UI gating reads cleanly and `canViewTagUsage = false` defaults at `UsageViewSelect.tsx:156` mean any consumer that hasn't been updated still gets safe (hidden) behavior. No regressions for downstream callers.

## Specific lines
- `litellm/proxy/_types.py:675-676` — adds two read routes to the internal-user allowlist. Correct scope.
- `litellm/proxy/_types.py:680-684` — extends `internal_user_view_only_routes` with the same two routes, so view-only role also gets through. Symmetric and correct.
- `UsagePageView.tsx:101` — `canViewTagUsage = isAdmin || internalUserRoles.includes(userRole || "")`. Reads correctly.
- `UsagePageView.tsx:1110-1115` — props threaded down to `UsageViewSelect`. Correct.
- `UsageViewSelect.tsx:163-165` — special-case `if (option.value === "tag") return isAdmin || canViewTagUsage;` before generic `adminOnly` filter. Correct order.
- `UsageViewSelect.tsx:148, 156` — new optional prop with safe `false` default. Correct.
- `test_route_checks.py:96-130` — parametrized 2×2 coverage. Good.

## Nits before merging
1. **PR is targeting a non-`main` branch (`litellm_internal_staging`).** The base branch name suggests this lands in an internal staging line first rather than `main`. Worth confirming with a maintainer that this is the intended integration path and that a forward-merge to `main` is tracked, otherwise the fix lives only on a side branch and upstream users on `main` keep hitting the bug.
2. **Hard-coded route strings appear in three places now** (`internal_user_allowed_routes`, `internal_user_view_only_routes`, and the test parametrize list). A shared `TAG_USAGE_READ_ROUTES = ["/tag/daily/activity", "/tag/list"]` constant in `_types.py` referenced by both allowlists — and imported into the test — would make the next route addition (e.g. `/tag/info`) a one-line change and make drift impossible.
3. **`try/except + pytest.fail` in `test_internal_users_can_access_tag_usage_read_routes`** at `test_route_checks.py:115-120` swallows the original traceback into the failure string. A plain call without the wrapper, or `pytest.raises(SomeError)` inverted with a `pytest.deprecated_call`-style helper, would surface the actual exception type/location on regression. Minor.
4. **Console-log noise:** `UsagePageView.tsx:98-99` shows two pre-existing `console.log` calls (`currentUser`, `currentUser max budget`) — not introduced by this PR but worth a separate cleanup, since `currentUser` may include PII (email, role, budget) and these logs ship to end-user browser consoles in production builds.
5. **Body references `$SHIN_GITHUB_TOKEN` and `/opt/cursor/artifacts/...`** — these are evidence-pipeline implementation details from the PR author's tooling, not anything reviewers can verify. Worth scrubbing from the PR body before merge so the public commit history stays clean.

None of the above blocks. The behavior change and test coverage are right.
