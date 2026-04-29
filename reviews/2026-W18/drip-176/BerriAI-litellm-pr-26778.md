# BerriAI/litellm#26778 — fix(ui): org_admin unable to view organization-wide usage data

- PR: https://github.com/BerriAI/litellm/pull/26778
- Head SHA: `4356daf1`
- Author: milan-berri
- Base: `litellm_internal_staging`
- Size: +68 / -? across 2 files

## Summary

Bug fix for #25493: `org_admin` users navigating to **Usage → Your
Organization Usage** see `$0.00`, `0 requests`, empty charts, and no
organization filter dropdown. Root cause is at
`ui/litellm-dashboard/src/app/(dashboard)/usage/page.tsx:11` where the
component hardcodes `organizations={[]}` instead of fetching real data.
Fix calls the existing `useOrganizations()` hook (which hits
`/organization/list`) and forwards the result to `UsagePageView`. Adds
two unit tests asserting both the populated and undefined-data branches.

## Notable design choices

- `page.tsx:13` — `const { data: organizations } = useOrganizations();`
  destructures only `data` (no `isLoading`, no `error`). The downstream
  `UsagePageView` handles the empty-array case (it just hides the
  filter dropdown), so there's no spinner needed. Acceptable for a
  small dashboard fix; the loading shimmer would be a separate UX
  improvement.
- `page.tsx:14` — `organizations={organizations ?? []}` keeps the
  `[]` fallback so `UsagePageView`'s array contract is preserved. Good
  defensive coalescing.
- `page.test.tsx:31-43` — `vi.hoisted()` for the three mock factories
  (`mockUseOrganizations`, `mockUseAuthorized`, `mockUseTeams`) is the
  correct vitest pattern when the mock identity needs to be referenced
  by both the `vi.mock()` call and the test body. Avoids the
  initialization-order trap that hits naive `vi.fn()` declarations.
- `page.test.tsx:54-65` — first test asserts both `useOrganizations`
  was called *and* the props passed to `UsagePageView` exactly match
  `{teams, organizations}`. Tight assertion, won't drift silently.

## Concerns

1. **PR base is `litellm_internal_staging`, not `main`.** This is a
   "rebased onto current litellm_internal_staging, cherry-picked
   community commit 90db2eb (#25617)" pattern. Reviewer should verify
   that #25617 (the upstream community PR being cherry-picked) is
   actually being credited, and that the `litellm_internal_staging` →
   `main` merge will preserve the cherry-pick author. If staging is
   squash-merged to main, the original author attribution gets lost.
2. **No test for the `org_admin` role specifically.** The
   `mockUseAuthorized` returns `userRole: "org_admin"` at
   `page.test.tsx:46`, but the test doesn't *assert* the role is
   forwarded or filtered against. If the bug recurs because
   `useOrganizations()` starts requiring a specific role and silently
   returns `[]` for `org_admin`, this test still passes. Add a third
   test where `useOrganizations` mock returns role-filtered data and
   assert `UsagePageView` gets it.
3. **`Pre-Submission checklist` first item ("Added testing in
   `tests/test_litellm/`") is unchecked.** Two tests *were* added but
   in `ui/litellm-dashboard/src/app/(dashboard)/usage/page.test.tsx`,
   not in the documented `tests/test_litellm/` location. Either update
   the checklist to acknowledge the UI test path is acceptable for
   UI-only fixes, or move the tests. As-is, a strict CI gate that
   greps for `tests/test_litellm/` additions would fail.
4. **Greptile review checkbox is unchecked.** Repo CONTRIBUTING flow
   asks for a `@greptileai` 4/5 confidence score before maintainer
   review. Author should run that pass.
5. **`UsagePageView` doesn't appear in the diff.** The fix assumes
   `UsagePageView` correctly handles a populated `organizations`
   array (renders the dropdown, filters charts, etc.). If that
   downstream component has its own bug — e.g., it expects
   `organization_alias` to be non-null and crashes on legacy orgs
   without one — this PR silently exposes it. Worth a manual smoke
   test on a tenant with mixed-quality org data.
6. **`useOrganizations()` import path
   `@/app/(dashboard)/hooks/organizations/useOrganizations` at
   `page.tsx:6`.** Confirm this hook does *not* require admin scope
   (`proxy_admin`) — if it does, the call will 403 for `org_admin`
   and the dropdown stays empty for the same reason as before, just
   with an extra failed network request.

## Verdict

**merge-after-nits** — clean targeted fix that does exactly what the
issue asks. The cherry-pick attribution (item 1) and `org_admin`-scoped
backend behavior of `useOrganizations()` (item 6) are the two items
worth verifying before merge — both are policy/integration issues that
the unit tests can't catch. Items 2-3 are checklist hygiene.
