# google-gemini/gemini-cli#26249 — fix(cli): hide read-only settings scopes

- **Head SHA:** `a74fa7016d3e6d499972d39f9876705f1d9f390f`
- **Author:** community
- **Size:** +256 / −27, 5 files

## Summary

The `/settings` dialog let users select read-only scopes (system, system-defaults, missing-path workspace). Editing such a scope only updated runtime state and silently failed to persist. This PR filters read-only scopes out of the editable list and adds a fallback to the first writable scope, plus preserves the `readOnly` field through `LoadedSettings.getSnapshot()`.

## Specific citations

- `settings.ts:404` — adds `readOnly: file.readOnly` to the cloned snapshot. Critical: without this the UI couldn't tell which scopes were read-only after a snapshot, since `getSnapshot()` was deep-cloning everything except this flag.
- `settings.test.ts:3006-3043` — new test `getSnapshot() should preserve readOnly metadata for each scope` — directly verifies the regression. Asserts `snapshot.system.readOnly === true`, etc.
- `SettingsDialog.tsx:111-122` — new `editableScopeItems` `useMemo` that filters via `settings.forScope(item.value).readOnly !== true` AND, for `Workspace`, requires `settingsFile.path !== undefined`. The path check correctly handles the home-directory edge case where the workspace scope is logically read-only because there's no project workspace file to write.
- `SettingsDialog.tsx:124-126` — `writableSelectedScope` falls back to `editableScopeItems[0]?.value` when the current `selectedScope` is filtered out. `effectiveSelectedScope` defaults to `SettingScope.User` if everything is read-only — see verdict.
- `SettingsDialog.tsx:128-132` — `useEffect` syncs state if `writableSelectedScope` differs. Correct pattern, no infinite loop because the effect only fires when the comparison flips.
- `SettingsDialog.tsx:233, 246, 257` — three call sites updated from `selectedScope` to `effectiveSelectedScope`. Dependency arrays updated to match (`:246`, `:262`).
- `SettingsDialog.test.tsx:609-732` — four new tests cover: (a) read-only system not offered, (b) home-dir workspace not offered, (c) fallback to writable, (d) no save when all scopes read-only. The fourth case is the important regression guard — `expect(setValueSpy).not.toHaveBeenCalled()` (`:730`).

## Verdict: `merge-as-is`

Tight, well-tested fix. The regression scenario, the snapshot fix, the UI filter, and the fallback are all addressed with a regression test for each. The test file is paired with a small `MockSettingsFileWithReadOnly` helper (`:84-95`) that's clean and reusable.

One minor observation, not blocking: `effectiveSelectedScope = writableSelectedScope ?? SettingScope.User` (`:126`) means if `User` is *also* read-only and there are no writable scopes at all, the dialog will still render against `User` and silently fail to save. The fourth test case actually exercises this — it asserts `setValueSpy not toHaveBeenCalled`, so behavior is "no-op rather than crash", which is reasonable. Could be improved to surface a "no writable settings scope" message to the user, but that's a follow-up.

Solid work. Ship it.
