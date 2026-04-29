# QwenLM/qwen-code#3491 — feat: add /diff command and git diff statistics utility

- PR: https://github.com/QwenLM/qwen-code/pull/3491
- Head SHA: `ecb7743c`
- Author: BZ-D (Edenman)
- Base: `main`
- Size: +3374 / -? across many files (core util + CLI command + i18n + 529-line test file)

## Summary

Implements the structured git-diff statistics feature requested in
#2997 and exposes it through a new `/diff` slash command.

- New core utility `packages/core/src/utils/gitDiff.ts` parses
  `git diff --numstat`, `--shortstat`, and unified `git diff HEAD`
  into a structured `GitDiffResult` (files changed, lines +/−,
  per-file summaries, hunks) with the issue's caps: 50 files, 1 MB
  per file, 400 lines per file, plus a 500-file short-circuit via
  `--shortstat`.
- New built-in `diffCommand` registered at
  `packages/cli/src/services/BuiltinCommandLoader.ts:104` between
  `copyCommand` and `deleteCommand`.
- 529-line test file
  `packages/cli/src/ui/commands/diffCommand.test.ts` plus core util
  tests (not visible in first 1500 lines of diff but implied by PR
  body).
- i18n strings added for `en`, `zh`, `zh-TW` covering header
  ("N files changed, +A / −R"), error states, and per-file
  annotations (binary/new/deleted/partial).

## Notable design choices

- **Linked worktree handling.** PR body calls out that `findGitRoot`
  returns the dir containing `.git`, but in a linked worktree `.git`
  is a *file* pointing at `<main>/.git/worktrees/<name>/`. New
  `resolveGitDir` helper follows that indirection so `MERGE_HEAD`,
  `CHERRY_PICK_HEAD`, etc. are looked up in the right gitdir. This
  is the kind of corner case that 80% of similar implementations
  get wrong — points for catching it.
- **Rebase detection.** PR body notes that `REBASE_HEAD` alone
  misses the common case (interactive rebase, am-rebase). The
  transient-state check looks at `rebase-merge/` and `rebase-apply/`
  directories too. Correct.
- **`---`/`+++`/`index` line content.** PR body acknowledges that
  the prior parser strategy (skip any line starting with those
  prefixes as header metadata) silently drops *content* lines that
  happen to start with those tokens — e.g., a Markdown file with
  `--- separator ---` would lose lines. New parser uses positional
  state (in-header vs in-hunk) instead of prefix matching.
- **i18n is exhaustive.** All user-facing strings have entries in
  `en`, `zh`, `zh-TW`. The plural-aware patterns
  `'{{count}} file changed, +{{added}} / -{{removed}}'` vs
  `'{{count}} files changed, +{{added}} / -{{removed}}'` are
  duplicated per locale — typical i18next pattern, slightly verbose
  but correct.
- **Test file is 529 lines for a single command.** Comprehensive
  coverage of `computeDiffColumnWidths`, mock `fetchGitDiff`
  behavior, and the rendering layer. The mock setup at
  `diffCommand.test.ts:14-22` cleanly stubs only `fetchGitDiff`
  while preserving the rest of the `@qwen-code/qwen-code-core`
  exports — good vitest pattern.

## Concerns

1. **PR size — +3374 lines.** Touches core utility, CLI command,
   builtin loader registration, three i18n locales, and a 529-line
   test file. Reviewable but not in one sitting. Author should
   consider splitting:
   - PR 1: core `gitDiff.ts` utility + core tests (the parser
     work, with all the worktree/rebase/`---` handling)
   - PR 2: `/diff` command wiring + i18n + UI tests
   This would let the parser get scrutinized on its own merits
   without being mixed with i18n string churn.
2. **No mention of test results in PR body's "Tests" section.**
   The PR description says "/Dive Deeper" with parsing details but
   doesn't paste `npm test` / `vitest` output. Adversarial review
   was claimed (mentions "issues surfaced during adversarial
   review") but the surface area is large enough that a CI green
   tick or pasted test output would help reviewers trust the
   change.
3. **"Borrowed the parsing strategy from the reference
   implementation in claude-code-cli's `utils/gitDiff.ts`"** — the
   PR body acknowledges the inspiration, which is good attribution.
   Verify the upstream license is compatible with qwen-code's
   license (claude-code-cli is Apache 2.0 IIRC; qwen-code is
   Apache 2.0; should be fine but a NOTICE/CREDITS reference is
   the conservative move).
4. **Caps (50 files, 1 MB/file, 400 lines/file, 500-file
   short-circuit) are hardcoded.** No env var, no config option.
   For a `/diff` command on a monorepo with a 600-file change,
   the user gets the `--shortstat` summary only — no way to
   override. Consider adding `/diff --all` or a config setting
   `diffStats.maxFiles` for power users.
5. **Per-file rendering format `+10 -2  src/utils/gitDiff.ts`.**
   The PR body shows three-column alignment (+N -N path) which is
   readable but `computeDiffColumnWidths` (exported from the
   command for testing) implies the alignment is dynamic. If a
   single file has +9999 -9999 deltas, the column blows up. Bound
   the column width and truncate with ellipsis if needed.
6. **`?` for untracked, `~` for binary at PR body's example.**
   These sigils aren't documented in any of the i18n strings. If
   a user pipes `/diff` output into something else, they need to
   know what the symbols mean. Add a legend or include the
   sigils in the i18n strings as `'{{count}} untracked'` /
   `'{{count}} binary'` annotations.
7. **`diffCommand` placement at
   `BuiltinCommandLoader.ts:104` between `copyCommand` and
   `deleteCommand`.** Alphabetical order would put `diff` between
   `delete` and `directory`. Fine either way, but the loader
   array isn't strictly sorted today, so this is a "match local
   convention" call.

## Verdict

**merge-after-nits** — substantive feature with thoughtful
handling of git's edge cases (linked worktrees, rebase states,
content lines that look like headers), clean i18n coverage, and
real test depth. The main asks before merge are: split the PR
(item 1) for cleaner review, paste test output (item 2), confirm
license attribution (item 3), and consider unbounded column
widths (item 5). The cap configurability (item 4) and sigil
documentation (item 6) can be follow-ups. As-shipped this is
useful but the PR is too big for confident single-pass review.
