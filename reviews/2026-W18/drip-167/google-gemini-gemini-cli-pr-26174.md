# google-gemini/gemini-cli PR #26174 — fix: remove instruction to combine shell commands with &&

- Repo: google-gemini/gemini-cli
- PR: https://github.com/google-gemini/gemini-cli/pull/26174
- Head SHA: `5354562b0b24839abd6511be37c8ffed92c2bc20`
- Author: zbynekwinkler (Zbyněk Winkler)
- Size: +0 / −2, 2 files
- Closes: #18022

## Context

The system prompt's git-repo guidance at
`packages/core/src/prompts/snippets.ts:499` and its legacy twin at
`packages/core/src/prompts/snippets.legacy.ts:368` instructed the agent to
"Combine shell commands whenever possible to save time/steps, e.g. `git
status && git diff HEAD && git log -n 3`." On Windows + PowerShell that
syntax is invalid (PowerShell uses `;` for sequential, `&&` only since
PS 7+ and not in the legacy `cmd.exe` PATH that some users still hit), so
the model would emit `&&`-joined commands, the shell would error, and the
model would have to retry with broken-out commands. The author's empirical
observation in the PR body — *"No amount of additional prompting can make
the model forget something it has in its context"* — is the right framing:
removing the offending line is more reliable than adding "but on Windows,
use `;` instead" instructions that the model then has to disambiguate.

## What the diff does

Two identical one-line removals:

```
- - Combine shell commands whenever possible to save time/steps, e.g. \`git status && git diff HEAD && git log -n 3\`.
```

at `snippets.legacy.ts:371` and `snippets.ts:502`. Nothing else changes —
the surrounding bullets about `git status` / `git diff` / `git log` style
are kept intact, as are the commit-message guidance lines that follow.

## Risks and counterpoints

The change is small and the rationale is sound, but there are a few
second-order questions:

1. **Cross-platform regression on macOS/Linux.** Removing the instruction
   doesn't *prevent* the model from chaining with `&&` on shells where
   it's valid — the author notes that in their test "it did combine
   commands even without the instructions and it used `;` to do so",
   which is a great signal that the model already knows to chain. But
   `;` semantics differ from `&&`: `;` runs the second command even if
   the first fails, while `&&` short-circuits. For a `git status && git
   diff HEAD && git log -n 3` flow that's harmless (read-only chained
   commands), but if the model generalizes the `;` chaining to a
   destructive sequence (`rm -rf foo ; mv bar foo`), the failure mode is
   worse than the chaining-failure on Windows. A replacement bullet —
   *"When chaining commands on POSIX shells prefer `&&` to short-circuit
   on first failure"* — preserves the intent while not pinning a
   specific syntax that breaks on Windows.

2. **No test pinning the prompt content.** Prompt regressions are usually
   silent; the next refactor that re-renders `renderGitRepo()` could
   re-introduce the line via merge conflict. A snapshot test asserting
   the rendered string does *not* contain `"&&"` would lock in the
   removal. Lightweight and high-leverage.

3. **Validation is single-developer empirical only.** The PR body says
   "I did use it for my day to day work for one whole workday". That's
   honest, but for a system-prompt change that affects every user on
   every platform, it would be worth a quick A/B against the existing
   prompt-eval harness if the project has one (gemini-cli does maintain
   eval suites in `packages/core` and similar). Otherwise the PR relies
   on the maintainer's judgment that the removal won't change behavior on
   tasks that already worked.

4. **Issue #18022 is the only stated motivator** — worth confirming the
   fix lands the close-tag rather than just-mentions, and that the issue
   has the same Windows-PowerShell context the PR body describes.

5. **`snippets.legacy.ts` parity.** The legacy file is presumably kept
   for backward-compat with older session shapes; the symmetric removal
   is correct. No drift.

## Suggestions

- Replace the deletion with a positive POSIX-shell guidance bullet ("On
  POSIX shells prefer `&&` to short-circuit on first failure") so the
  model retains the short-circuit semantics on platforms where it
  applies, while no longer being told to use `&&` unconditionally.
- Add a snapshot test against `renderGitRepo({...})` asserting the
  rendered output does not contain `"&&"` to prevent silent regressions.
- Consider a `process.platform === 'win32'` branch in `renderGitRepo`
  that emits `;` guidance on Windows and `&&` guidance elsewhere — this
  is the most surgical fix but adds complexity.

## Verdict

**merge-after-nits** — removal is correct and the rationale matches the
empirical evidence. The nit is the missing snapshot test and the
question of whether to replace the line with platform-specific guidance
versus dropping it entirely.
