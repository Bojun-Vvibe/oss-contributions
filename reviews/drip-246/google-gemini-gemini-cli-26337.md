# PR #26337 — fix(ci): robust version checking in release verification

- Repo: google-gemini/gemini-cli
- Author: scidomino
- Head: `a66b66ea3cc0d98f61b46415a0ebcc6f1df78339`
- URL: https://github.com/google-gemini/gemini-cli/pull/26337
- Verdict: **request-changes**

## What lands

Two-line change in `.github/actions/verify-release/action.yml:66,83`
adding `2>/dev/null` to the version-capture commands:

```diff
-        gemini_version=$(gemini --version)
+        gemini_version=$(gemini --version 2>/dev/null)
...
-        gemini_version=$(npx --prefer-online "${INPUTS_NPM_PACKAGE}" --version)
+        gemini_version=$(npx --prefer-online "${INPUTS_NPM_PACKAGE}" --version 2>/dev/null)
```

## Specific findings

- The intent appears to be: `gemini --version` (or `npx ... --version`) is
  emitting noise on stderr (deprecation warnings, npx download progress,
  etc.) which is bleeding into the variable capture or log. Suppressing
  stderr makes the captured stdout clean.
- Problem: `$()` only captures stdout. Even pre-PR, stderr was *not* being
  captured into `$gemini_version`. So the PR is not fixing variable
  contamination — it's only suppressing the *display* of stderr in the
  CI log.
- That is exactly the wrong direction for "robust version checking in
  release verification." If `gemini --version` is failing (binary not
  found, broken install, network error during npx fetch), the diagnostic
  the on-call needs is on stderr. Now it's silently swallowed. The next
  failure mode — empty `$gemini_version` triggers the
  "❌ NPM Version mismatch: Got  from ... expected ..." branch at `:67-69`
  — now has no diagnostic to root-cause from.
- Correct fix shape:
  - If the goal is "don't pollute the log with non-fatal stderr noise":
    `gemini_version=$(gemini --version 2>/tmp/gemini-stderr)` and then
    only `cat /tmp/gemini-stderr` if the version-mismatch branch fires.
  - If the goal is "capture stderr too because it might contain the
    version string": `gemini_version=$(gemini --version 2>&1)`.
  - If the goal is "fail loud when the command itself errors":
    `set -euo pipefail` at the top of the run block (also forces the
    `if` check to compare against a guaranteed-set variable).
- The PR title says "robust version checking" but the change *reduces*
  observability without adding any robustness. No `set -e`, no exit-code
  check on the version-emitting command, no fallback if stderr was the
  carrier. A truly robust version check would also assert that
  `$gemini_version` is non-empty before the comparison at `:67`/`:84`,
  which currently happily compares `""` against `${INPUTS_EXPECTED_VERSION}`
  and emits the unhelpful `Got  from ...` error.

## Risks

- Future CI failures lose root-cause information. On-call has to re-run
  locally without the redirect to see what `gemini --version` was actually
  saying.
- Sets a precedent that `2>/dev/null` is "robustness" — that pattern will
  spread to other CI steps and degrade the diagnostic surface.

## Verdict

**request-changes**. The two-character fix is the inverse of robust:
it discards diagnostic signal at the exact moment the script is checking
for a failure mode where diagnostic signal is most needed. Ask the author
to either (a) capture stderr to a tempfile and emit it on mismatch, (b)
add `set -euo pipefail` and an explicit empty-string check, or (c)
combine stderr into the version capture with `2>&1` if that's what was
actually observed in the failing run.
