# google-gemini/gemini-cli PR #26317 — fix: bypass powershell command substitution check for setup-github

- URL: https://github.com/google-gemini/gemini-cli/pull/26317
- Head SHA: `b6e716b370e5f4c203afd6d3e453b9481c80fc15`
- Author: cloudwaddie-agent
- Repo: google-gemini/gemini-cli

## Summary

Hardens `/setup-github` against shell injection by switching ad-hoc
double-quoting to `escapeShellArg(..., shell)` per the active
`getShellConfiguration()`, validates the GitHub release tag against a
strict semver regex, and removes the outer subshell wrapper.

## Specific notes

- `packages/cli/src/ui/commands/setupGithubCommand.ts:26-30` — pulls
  `escapeShellArg` and `getShellConfiguration` from
  `@google/gemini-cli-core`; correct provenance, no new dep.
- `setupGithubCommand.ts:67-72` — URL escaping moves from
  `${openCmd} "${url}"` to
  `${openCmd} ${escapeShellArg(url, shell)}`. With this, a release tag
  containing `"` or `$` (PowerShell) or backticks (POSIX) can no longer
  break out of the open command. This is the substantive security fix.
- `setupGithubCommand.ts:243-247` — release tag validated by
  `/^v?\d+\.\d+\.\d+(-[a-zA-Z0-9.]+)?$/`. Tight enough to block injected
  metadata. Note: prerelease segment doesn't allow `+build` metadata
  (semver `+`); GitHub release tags rarely include `+`, so this is fine
  in practice but should be documented if anyone pins a `v1.2.3+ci.42`
  tag in their fork.
- `setupGithubCommand.ts:275-280` — the `echo "..."` line is replaced
  with `echo ${escapeShellArg(echoMessage, shell)}`. Good: the readme
  URL containing the (now-validated) release tag is interpolated into
  the message safely.
- `setupGithubCommand.ts:282` — drops the outer `(${commands.join(' &&
  ')})` parenthesization. Verify this doesn't change `set -eEuo
  pipefail` scoping on POSIX: bare `&&` chain still propagates failures
  but `set -e` now persists in the parent shell. For `run_shell_command`
  this is a fresh sub-shell anyway, so impact is nil — but worth a
  one-line comment explaining why the parens were removed.
- No new tests. Recommend at minimum a unit test feeding a malicious
  release tag like `v1.0.0";rm -rf /;echo "` and asserting the regex
  rejects it.

verdict: merge-after-nits
