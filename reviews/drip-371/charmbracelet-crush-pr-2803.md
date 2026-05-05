# charmbracelet/crush PR #2803 — bug: yollo mode via flag doesn't activate prompt

- URL: https://github.com/charmbracelet/crush/pull/2803
- Head SHA: `fd5f9301283778a6dc09a27bab65087077b018d0`
- Author: taciturnaxolotl (Kieran Klukas)
- Fixes: #2802
- Files: `internal/ui/model/ui.go` (+1/-1)
- Reviewer status: meowgorithm (Approved)

## Assessment

One-line fix for a real UX bug. The change at `internal/ui/model/ui.go:350` replaces a hardcoded `ui.setEditorPrompt(false)` with `ui.setEditorPrompt(com.Workspace.PermissionSkipRequests())`. The previous code always initialized the editor prompt as if "yollo" (skip-permission-requests) mode was off, even when the user had passed the `--yolo` flag at startup — so the visual prompt indicator stayed in safe-mode appearance until the user toggled it manually inside the session. The fix reads the actual workspace permission state (which is populated from CLI flags + config at startup) so the prompt UI matches the actual security posture from the first frame.

The fix is the minimum-viable correct change: it consults the same `Workspace.PermissionSkipRequests()` accessor that other parts of the system already use to gate destructive actions, so the prompt indicator can never drift out of sync with the actual permission state. There's no risk of a false positive (the accessor returns the canonical state), and no risk of breaking the in-session toggle flow because `setEditorPrompt` can still be called later with the new value when the user toggles.

`meowgorithm` (a charmbracelet maintainer and one of the project's primary reviewers) has already approved per the PR metadata. Combined with the trivial diff size and the clear linkage to a filed bug (#2802), there's no reason to hold this. Worth confirming the fix is also covered by either a manual smoke-test note in the PR or an integration test at the `ui` package level — the diff doesn't show a new test, which is fine for a one-line UI initialization fix where a unit test would essentially be a tautology against the same accessor call. If the project has e2e CLI tests, a `crush --yolo` startup snapshot would be a nice follow-up but not blocking.

## Verdict

`merge-as-is`
