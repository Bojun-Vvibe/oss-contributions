# charmbracelet/crush PR #2702 — feat: super yollo (dangerous-command warning in YOLO mode)

Link: https://github.com/charmbracelet/crush/pull/2702
State: OPEN · Author: taciturnaxolotl · +378 / −37 · Files: 16

## Summary
Adds a "dangerous command" classification to the bash tool. In YOLO mode, dangerous commands
(curl, sudo, package managers, `npm install --global`, `pip install --user`, `go test -exec`,
brew, etc.) are no longer silently auto-approved — they raise a permission prompt with a
warning banner. Previously *blocked* commands now run after explicit approval rather than
returning an error. Non-dangerous commands remain auto-approved in YOLO mode. A new
"super yollo" flag bypasses the prompts to restore the prior YOLO behavior. The `Dangerous`
flag is plumbed through the permission service, the request type, the dialog UI, the
template, and a new `PermissionMode` accessor on the service.

## Strengths
- The default shifts from "silent execute" to "prompt with warning", which is the safer
  default for an "auto-approve everything" mode that historically silently ran `curl |
  bash`-class commands. Reasonable fail-safe direction.
- New `TestIsDangerousCommand` table covers 12 representative commands across npm, pip,
  brew, sudo, curl, go test, with both dangerous and adjacent-safe variants. Good
  classifier coverage for the moments that matter.
- The escape hatch (super-yollo) is opt-in and named such that its danger is unmistakable.
  Users who explicitly want the old behavior can have it without the project carrying the
  blame for the default.

## Concerns / Risks
- **Block-list-as-classifier is fragile.** The dangerous classifier is `shell.IsCommandBlocked`
  applied to a static `blockFuncs()` set. Trivial bypasses include: `bash -c "curl ..."`,
  `eval "$(echo curl ...)"`, command substitution `$(curl ...)`, aliases, or simply piping
  through a wrapper script (`./run.sh` where `run.sh` calls curl). The PR description doesn't
  claim defense in depth, but if the safety story leans on this classifier for a real
  threat-model boundary, those gaps will bite. The classifier is fine as a UX nudge; calling
  it "security" would overstate it.
- **`bgManager.Start` no longer receives `blockFuncs()`** — the call sites pass `nil`. The
  comment says "permission check handles dangerous commands". That is true *only* for the
  approval branch (`!isSafeReadOnly` → `permissions.Request`). The unconditional second
  `bgManager.Start` call (line 311) also passes `nil`. If any code path reaches that second
  Start without going through the permission check first, the previous block-functions
  defense is gone. Worth a code-path audit.
- **Edge case: `isSafeReadOnly && isDangerous`.** Can a command be both? If the safe-read-only
  classifier ever overlaps with the dangerous list (e.g., a `git` subcommand that reads but
  also fetches over the network), the dangerous branch is skipped entirely. There is no
  test crossing these two classifiers.
- **UI / dialog rendering of the warning banner is untested in this PR.** The `Dangerous`
  flag is added to the request type and threaded through `permissions.go`,
  `dialog/permissions.go`, and `styles.go`, but there are no UI snapshot or model tests for
  the new banner appearance. A regression where the banner silently fails to render would
  reduce dangerous commands back to plain prompts.
- **Mock interface drift.** Three test mocks now implement `SetPermissionMode` and
  `PermissionMode`. Any third-party consumer embedding `permission.Service` will break at
  compile time on upgrade. Worth a release-notes line.
- **No `super_yollo` test asserting it actually bypasses the dangerous prompt.** The opt-in
  is the most dangerous code path in this PR; a test pinning that "super_yollo + dangerous
  command = no prompt" prevents a future rename from accidentally enforcing prompts even
  in super-yollo.

## Suggested follow-ups
1. Add a regression test that walks `bash -c "curl ..."` and similar wrappers through the
   classifier, then either tighten the classifier or document the limitation in the user
   docs.
2. Add a snapshot test for the warning-banner dialog rendering.
3. Add a behavioral test pinning `super_yollo = true` skips dangerous prompts; complement
   it with one pinning `super_yollo = false` raises them.
