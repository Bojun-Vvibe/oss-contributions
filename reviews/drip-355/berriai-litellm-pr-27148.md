# BerriAI/litellm PR #27148 — fix(ui): omit unchanged allowed_routes on key update

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/27148
- Head SHA: `31f95d9117cc`
- Size: +118 / -0 across 2 files (1 component, 1 test)
- Fixes upstream #27005.

## Summary

Non-admin team admins editing a key in the UI were tripping a
backend permission check on `allowed_routes` even when they hadn't
touched the field — the form was always serializing
`allowed_routes` into the PATCH body, and the backend (correctly)
treats any non-null value on that field as "you are setting
allowed_routes, do you have permission?". This PR makes the UI
strip `allowed_routes` from the payload when the submitted value is
semantically equal to the existing key's value, so a no-op save is
also a no-op write.

## What I like

- The "what is unchanged?" predicate is a reusable normalizer.
  `key_info_view.tsx:52-66` adds `normalizeStringList(value)` that
  accepts arrays *or* comma-separated strings (the form uses both
  shapes depending on widget) and returns a trimmed, non-empty
  string list. `areStringListsEqual` (lines 68-75) compares two
  normalized lists element-by-element. Clear, side-effect-free, no
  `JSON.stringify` shortcut (which would silently fail on whitespace
  / order).
- The strip happens at one place, `key_info_view.tsx:200-204`, with
  a load-bearing comment:
  ```
  // Strip unchanged allowed_routes so non-admin editors don't trip the
  // backend "setting allowed_routes" permission check on a no-op save.
  if (areStringListsEqual(formValues.allowed_routes, currentKeyData.allowed_routes)) {
    delete formValues.allowed_routes;
  }
  ```
  Future maintainers will know exactly *why* this delete exists
  (which is the failure mode the prior code lacked).
- The test file
  `ui/litellm-dashboard/src/components/templates/key_info_view.test.tsx`
  adds three cases covering the realistic matrix:
  - same array → drop (lines 808-826, asserts
    `expect.not.objectContaining({ allowed_routes: ... })`)
  - empty input on a key that previously had no override → drop
    (lines 828-845)
  - empty input on a key that *did* have an override → keep
    (the user is intentionally clearing it; lines 847-863 assert
    `expect.objectContaining({ allowed_routes: [] })`)
  That last case is the important one — easy to over-strip and
  silently swallow a clear-action.
- The role-mock harness (`enterEditMode` helper at lines 779-797)
  forces `userRole: "proxy_admin"` so the test is exercising the
  real authorization scenario, not a privileged shortcut.

## Nits / discussion

1. **Comparison ordering is order-sensitive.**
   `areStringListsEqual` compares element-by-element after trim
   (`normalizedLeft.every((entry, index) => entry === normalizedRight[index])`).
   If the form widget reorders entries (e.g. drag-to-reorder) but
   the user didn't add/remove any, this returns `false` and we'd
   PATCH a "no-op reorder" through the permission check. If route
   order has no semantic meaning on the backend, an order-
   insensitive set comparison would be safer. Worth confirming
   with the backend semantics.

2. **Only `allowed_routes` is treated this way.** The same "no-op
   PATCH trips a permission check" pattern almost certainly exists
   for other admin-only key fields (rate limits, model lists, team
   membership). This PR is the right shape; it just doesn't
   generalize to a `stripUnchangedFields(formValues, currentKeyData,
   sensitiveFields)` helper. Worth a follow-up issue: "audit other
   permission-gated fields for the same no-op-save bug."

3. **`formValues` is mutated in-place.** `delete formValues.allowed_routes`
   modifies the caller's form-state object. Probably fine because
   submit-handlers usually treat `formValues` as ephemeral, but if
   anything downstream re-reads `formValues.allowed_routes` (e.g.
   for an "are you sure?" diff modal), it'll now see `undefined`
   instead of the user-typed value. A `const payload = { ...formValues
   }; delete payload.allowed_routes;` pattern would be safer.

4. **Test duplication.** The three new tests share a lot of
   `enterEditMode` + `userEvent` boilerplate. A small `submitWith`
   helper (or a `it.each` table) would shrink the diff. Cosmetic.

5. **Server-side belt-and-suspenders.** The real fix is "don't gate
   no-op writes on permission" — i.e. the backend should also accept
   an unchanged `allowed_routes` from a non-admin. This UI fix
   addresses the user-visible symptom but doesn't close the loophole
   for non-UI clients (CLI, automation) hitting the same endpoint.
   Worth a follow-up.

## Verdict

**merge-after-nits.** Correct, focused fix for a real user-visible
permission bug, with three pointed tests including the easy-to-
miss "user is clearing an existing override" case. Comment at the
strip site is excellent. Worth confirming whether route order is
significant before considering this fully closed; otherwise a sort-
based comparison would be more robust. The server-side fix is the
proper long-term answer but doesn't block this UI patch.
