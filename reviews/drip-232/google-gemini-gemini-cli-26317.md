# google-gemini/gemini-cli#26317 — fix: bypass powershell command substitution check for setup-github

- **Author:** cloudwaddie (cloudwaddie-agent)
- **Head SHA:** `b6e716b370e5f4c203afd6d3e453b9481c80fc15`
- **Base:** `main`
- **Size:** +19 / -6 across 1 file
- **Files changed:** `packages/cli/src/ui/commands/setupGithubCommand.ts`

## Summary

Closes a real Windows-only failure of the `/setup-github` slash command where the generated shell command was wrapped in a parenthesized subshell expression `(cmd1 && cmd2 && ...)` that the CLI's own `detectCommandSubstitution` heuristic (designed to block injection-shaped command-substitution attempts in PowerShell) was flagging as suspicious and refusing to execute. Fix is two-axis: (a) drop the unnecessary subshell wrapper so the generated command no longer looks like an attack pattern; (b) properly escape URL arguments and the echo message via `escapeShellArg` instead of relying on naive double-quoting that breaks under PowerShell's quote-eating rules.

## Specific code references

- `setupGithubCommand.ts:26-30`: import widened from just `debugLogger` to also pull in `escapeShellArg` and `getShellConfiguration` from `@google/gemini-cli-core`. These are the two existing helpers the PR uses for the platform-aware escaping — correct discovery of pre-existing utilities rather than reinventing them.
- `setupGithubCommand.ts:67-72`: in `getOpenUrlsCommands(readmeUrl)`, old shape was `urlsToOpen.map((url) => \`${openCmd} "${url}"\`)` — naive double-quote wrap that fails on URLs containing inner double quotes (rare for URLs but the wider issue is PowerShell's interpretation of unbalanced quotes). New shape is `urlsToOpen.map((url) => \`${openCmd} ${escapeShellArg(url, shell)}\`)` — uses the platform-detected shell (bash, sh, pwsh, cmd) to pick the right escaping convention. The `getShellConfiguration()` call at the top of the function is the correct hoist (one shell-detection per invocation, reused for all URLs).
- `setupGithubCommand.ts:243-247`: new defensive validation arm — `if (!/^v?\d+\.\d+\.\d+(-[a-zA-Z0-9.]+)?$/.test(releaseTag)) { throw new Error(...) }` rejects malformed `releaseTag` values from `getLatestGitHubRelease(proxy)` before they get interpolated into the `readmeUrl` string. This is a real second-order security fix — without it, a compromised or buggy GitHub-release endpoint could return a tag like `'; rm -rf /` and have it land in the generated shell command. The regex correctly accepts canonical `vX.Y.Z` and `X.Y.Z-prerelease` shapes while rejecting anything with shell metacharacters.
- `setupGithubCommand.ts:275-277`: the previously-naive `commands.push(\`echo "Successfully downloaded ...${readmeUrl}..."\`)` becomes `commands.push(\`echo ${escapeShellArg(echoMessage, shell)}\`)` — the entire echo message (which embeds `readmeUrl`, which embeds `releaseTag`) now goes through the shell-aware escaper. Combined with the regex validation at `:243`, this closes the injection vector cleanly: the regex prevents weird input, and the escaper handles the well-formed-but-quote-containing case.
- `setupGithubCommand.ts:280`: the load-bearing fix to the actual reported failure — `const command = \`(${commands.join(' && ')})\`` becomes `const command = commands.join(' && ')`. The subshell wrapper was redundant for sequencing (`&&` already chains the commands left-to-right) but was the trigger for `detectCommandSubstitution`'s false positive on PowerShell. The new shape is functionally equivalent for the && chain and no longer trips the safety heuristic.

## Reasoning

This is a small but well-shaped PR that solves three things in one pass: (a) the immediate user-visible bug (PowerShell rejecting the generated command); (b) latent quoting hazards in the URL-open and echo paths that would have surfaced eventually under unusual URLs; (c) a real injection vector via the GitHub-release-tag input that wasn't previously validated. Each fix is at the right boundary: the wrapper-removal is at the command-construction site, the escaping is at the per-arg interpolation site, the validation is at the input-trust boundary.

Two notes worth maintainer attention: (a) the regex `^v?\d+\.\d+\.\d+(-[a-zA-Z0-9.]+)?$` correctly rejects shell metacharacters but doesn't accept `+build` SemVer build metadata (`1.2.3+build.456`) — if any historical release tag uses that shape it will now be rejected hard with the new error. Worth a quick `git tag -l` sanity check on the upstream `google-github-actions/run-gemini-cli` tag history. (b) The PR title says "bypass" but the actual fix is "no longer trigger" — the heuristic isn't bypassed, it just isn't triggered because the offending pattern is gone. Slight rename for clarity in the merge commit would help future archaeologists.

## Verdict

**merge-as-is**
