# BerriAI/litellm #26746 — fix(ui): add Your Usage view for admin users on usage page

- **PR:** https://github.com/BerriAI/litellm/pull/26746
- **Head SHA:** `a3a5250424fee9e43914c7cb2eb468cbdff2222a`
- **Size:** +13 / −6 (2 source files + 4 screenshot binaries)

## Summary
Re-files closed PR #26740. Restores admin-user access to a personal-usage view on the dashboard `/usage` page by adding an admin-only `"my-usage"` option to the `UsageOption` union, threading it through `UsagePageView.tsx`'s `effectiveUserId` resolver, and gating the "Filter by user" dropdown so it only renders in the global view.

## Specific observations

1. **The load-bearing line is `UsagePageView.tsx:172`** — `const effectiveUserId = isAdmin ? selectedUserId : userID || null;` becomes `const effectiveUserId = usageView === "my-usage" || !isAdmin ? userID || null : selectedUserId;`. Logic is correct: in `my-usage` mode (regardless of role) and non-admin mode, fall back to the logged-in user's `userID`; otherwise honor the admin's selected filter. The condition order matters — `usageView === "my-usage"` is checked first so an admin in `my-usage` mode correctly gets their own data even though `isAdmin` is true. Good.

2. **Panel-render gate widening at `:480-483`** — `usageView === "global"` becomes `(usageView === "global" || usageView === "my-usage")`, and the user-filter dropdown is wrapped in `isAdmin && usageView === "global"` so admins still see the dropdown in global mode but not in my-usage mode. Consistent with the `effectiveUserId` resolver. Worth verifying no downstream `<Text>` headers in the rendered panel (e.g. "Showing data for all users" or similar) read `selectedUserId` directly — if they do, an admin in my-usage mode could see a label that says "all users" while looking at their own data.

3. **`UsageViewSelect.tsx:14` extends the `UsageOption` union** with `"my-usage"` and adds a new entry at `:43-49` with `adminOnly: true` and `UserOutlined` icon. The option config follows the existing `OptionConfig` shape (`value`, `label`, `description`, `icon`, `adminOnly`). Good. The option is placed second in the array (right after `"global"`) — this is the right ordering for the dropdown UX (admins read top-down: global → mine → org → team → ...).

4. **Zero test coverage, despite the project template's "at least 1 test is a hard requirement" mentioned in PR #26740's review.** The change is small enough that a one-test smoke is reasonable: render `<UsagePageView>` with `isAdmin=true, usageView="my-usage", userID="admin-1", selectedUserId="user-2"`, assert `effectiveUserId` resolves to `"admin-1"`, and assert the user-filter dropdown is not rendered. This was raised in the prior closed iteration; it should be addressed before re-merging.

5. **The 4 screenshot PNG binaries under `.github/screenshots/`** are useful for PR review but bloat the repo's `.git`. The team's convention should decide between `.github/screenshots/` (kept) vs `https://github.com/user-attachments/...` upload-only (which is what the PR body links to with placeholder URLs). Two storage paths for the same screenshots is wasteful.

## Verdict: `merge-after-nits`

Logic is correct and minimal, but the missing tests are a recurring template-violation issue and the screenshot-binary storage location is inconsistent with the PR-body link convention.

## Recommended actions
- **(Required)** Add a unit test in `UsagePageView.test.tsx` (create if missing) covering the `effectiveUserId` resolver across the four `(isAdmin, usageView)` combinations: `(true, "global")`, `(true, "my-usage")`, `(false, "global")`, `(false, "my-usage")`. The third case is what regressed in the original bug report and is exactly what `usageView === "my-usage" || !isAdmin` is supposed to lock down.
- **(Verify)** Grep the rendered panel components for direct `selectedUserId` reads in user-facing strings — confirm no "showing data for X" header would mislabel admin's own data as "all users" in my-usage mode.
- **(Nit)** Pick one screenshot storage path (`.github/screenshots/` *or* user-attachments URLs) and stick with it — current PR has both.
