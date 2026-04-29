# BerriAI/litellm PR #26740 — fix(ui): add Your Usage view for admin users on usage page

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26740
- Head SHA: `5da8d6cb66`
- State: OPEN, +13/-6 across 2 files

## What it does

Restores admin users' access to their own personal-usage view on the LiteLLM dashboard's usage page. Pre-PR an admin (`proxy_admin` role) saw only the global usage page and had to manually search for themselves in the user-filter dropdown to see their own activity. This PR:

1. Adds a new `"my-usage"` value to the `UsageOption` union type in `UsageViewSelect.tsx:14`.
2. Adds an admin-only `"Your Usage"` option to the select dropdown (`OPTIONS` array, `:46-52`) using `UserOutlined` icon, with `adminOnly: true`.
3. In `UsagePageView.tsx:172` updates the `effectiveUserId` derivation: from `isAdmin ? selectedUserId : userID || null` to `usageView === "my-usage" || !isAdmin ? userID || null : selectedUserId`. So when an admin selects "Your Usage", `effectiveUserId` is forced to their own `userID` regardless of the user-filter dropdown's `selectedUserId` value.
4. Updates the panel render gate at `:480` from `usageView === "global"` to `usageView === "global" || usageView === "my-usage"` so the Your-Usage panel renders the same component tree as Global.
5. Hides the "Filter by user" dropdown when in `my-usage` view (`:483`: `isAdmin && usageView === "global"`).

## Specific reads

- `UsageViewSelect.tsx:14` — `UsageOption` union extension. Order-stable, additive — no other consumer of `UsageOption` should break since they'd type-check against the wider union. Verify no `switch (option) { case ... default: never }` exhaustiveness check exists elsewhere that would need a new arm. (Quick-grep for `as never` against `UsageOption` would catch it.)
- `UsageViewSelect.tsx:46-52` — option config. `adminOnly: true` is consumed elsewhere (existing pattern) — assuming the rendering layer respects that flag, non-admin users won't see the option. Worth a snapshot test that `OPTIONS` filtered for non-admin doesn't include `my-usage`.
- `UsagePageView.tsx:172` — the conditional rewrite is a small but load-bearing change. Read carefully:
  - **Pre**: `isAdmin ? selectedUserId : userID || null` — admin sees whatever `selectedUserId` resolves to (often `null` initially → global), non-admin sees their own.
  - **Post**: `usageView === "my-usage" || !isAdmin ? userID || null : selectedUserId` — admin in `my-usage` mode sees their own, admin in any other mode sees `selectedUserId`, non-admin always sees their own.
  
  The branch is correct. One subtle behavior change: pre-PR an admin in `global` view with `selectedUserId === null` would resolve to `null` (= "all users"). Post-PR, same admin in `global` view still resolves to `selectedUserId` which is `null`. Equivalent. ✓
- `UsagePageView.tsx:480-485` — panel render gate now includes `my-usage`. The `<>` fragment that previously wrapped the global panel and the user-filter dropdown is reused. The user-filter dropdown render (`isAdmin && usageView === "global"`) correctly hides for `my-usage`, so the admin sees their own usage panel without the "Filter by user" UI noise.
- **Missing**: no test surface added. The PR description claims "Adding at least 1 test is a hard requirement" per the LiteLLM PR template, but the diff is +13/-6 across two `.tsx` files with no `.test.tsx` companions. Maintainer policy will likely block on that.
- **Missing**: the PR doesn't change `UsagePage`'s passing of `selectedUserId` into the panel — only the `effectiveUserId` derivation. Confirm there's no second site downstream that reads `selectedUserId` directly and now disagrees with `effectiveUserId` in `my-usage` mode (e.g. a "showing data for user X" label that still says "all users" because it reads `selectedUserId`).
- **Missing**: no e2e check that the URL/query-param survives a `my-usage` selection. If the page persists view-state in URL, `?view=my-usage` should round-trip; if not, a refresh drops back to global. Minor UX nit.

## Risk

1. **Test gap** vs explicit project policy. PR template says tests are hard-required; this has none. Maintainer will request.
2. **Label drift** — confirm any "Showing data for: X" header in the right-hand panels reads from `effectiveUserId` not `selectedUserId`, otherwise admin in `my-usage` view sees their data with a "Showing all users" label. The diff doesn't touch any such header so by reading-only it's probably fine, but worth a manual smoke screenshot.
3. **`adminOnly` flag enforcement** is implicit — depends on the existing select-rendering layer to filter on the flag. If a non-admin somehow lands on `?view=my-usage` (URL persistence), the `UsagePageView` render still treats them as `!isAdmin`-branch (correctly forcing their own userID), so no privilege escalation. Confirmed safe.
4. **Translation/i18n**: hard-coded `"Your Usage"` and `"View your own usage"` strings. If LiteLLM dashboard supports i18n elsewhere, these need wrapping; if it's English-only, fine.

## Verdict

`merge-after-nits` — correct, minimal fix for a real admin-UX hole. Two structural nits: add at least one render test (admin sees option, non-admin doesn't; `effectiveUserId` resolves to admin's own userID when `usageView === "my-usage"`), and verify no downstream "showing data for X" label reads `selectedUserId` directly. The +13/-6 diff size is appropriate for the bug class.

## What I learned

The "admin sees nothing about themselves by default" anti-pattern shows up a lot in observability dashboards — the admin role is power-user enough that the UX assumes they want the bird's-eye view, but the natural human first instinct on a usage dashboard is "what about me?" The right fix isn't "add a hidden URL param" or "remember last selected user" — it's an explicit option in the view-selector that says exactly what it does. This PR gets that shape right; the gap is just the test coverage required by the project's stated policy.
