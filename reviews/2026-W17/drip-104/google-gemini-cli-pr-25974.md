---
pr: https://github.com/google-gemini/gemini-cli/pull/25974
sha: 196dc29e
diff: +21/-0
state: OPEN
---

## Summary

13-line behavior fix in `ThemeManager.findThemeByName` plus an 18-line regression test, restoring the ability to look up a file-loaded theme by its **internal display name** (the `name` field inside the JSON theme file) rather than only by its absolute filesystem path. Symptom: after a user did `setActiveTheme("/home/user/my-theme.json")`, subsequent `findThemeByName("My File Theme")` returned `undefined` even though the theme was loaded and active.

## Specific observations

- `packages/cli/src/ui/themes/theme-manager.ts:658-662` — adds a linear scan over `this.fileThemes.values()` matching by `theme.name`, placed between the existing path-based `fileThemes.get(themeName)` lookup at `:655` and the final `return undefined` at `:665`. Order is correct: literal-key lookup is O(1) and keeps fast-path priority, the new internal-name scan is O(n) and only fires on miss. For the typical case of <10 file-loaded themes per user this is unmeasurable; for a theme power-user with hundreds it would still be sub-millisecond.
- The scan returns the **first** match by insertion order — if two file-loaded themes happen to declare the same internal `name`, the second one is silently shadowed. Worth a one-line comment acknowledging this is a documented quirk (or alternatively returning `undefined` on ambiguity to match the path-lookup invariant of "exactly one theme per key"). Given the path key in `fileThemes` is the absolute filesystem path, two themes with the same `name` field would currently both load successfully under different keys, so the ambiguity is reachable in practice.
- `packages/cli/src/ui/themes/theme-manager.test.ts:155-168` — the new test correctly stubs `fs.existsSync` and `fs.readFileSync`, calls `setActiveTheme('/home/user/my-theme.json')` to seed `fileThemes`, then asserts `findThemeByName('My File Theme')` returns a theme with `name: 'My File Theme'`. Uses the `mockTheme` fixture defined elsewhere in the file. Pin is the minimum sufficient test — does not cover the ambiguity case mentioned above, nor the negative case (`findThemeByName('Nonexistent')` returns `undefined` after the new scan is added).
- The fix doesn't expose a new public API surface — `findThemeByName` already existed and is the public lookup primitive used by `/theme` command resolution and config reload, so this is the right insertion point rather than adding a parallel `findFileThemeByName`.

## Verdict

`merge-after-nits` — correct minimal fix at the right call site with paired regression test. Two cheap follow-ups before merge: (1) decide on the duplicate-internal-name policy (first-wins via comment, or undefined-on-ambiguity matching the rest of the lookup chain's invariants), (2) add a one-line negative test that `findThemeByName('does-not-exist')` returns `undefined` so a future blanket-match bug in the new scan doesn't go silent.
