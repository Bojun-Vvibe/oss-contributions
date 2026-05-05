# sst/opencode PR #25860 — fix(project): keep bare worktree roots checked out

- Head SHA: `4780710c54c7ba3b7b9c13e7861daa2e5022a247`
- Author: `osamu2001`
- Size: +7 / -4
- Verdict: **merge-as-is**

## Summary

Fixes a regression in `Project.fromDirectory` where opencode invoked
from inside a worktree of a bare-repo layout would set
`project.worktree = barePath` (i.e. the bare `.git` directory itself)
instead of `project.worktree = worktreePath` (the actual checked-out
working tree). Downstream code consuming `project.worktree` as the cwd
for git operations / file watchers / sandbox roots therefore pointed
at a directory with no working tree, breaking any
read/write/index op that wasn't pure plumbing.

## What the diff actually does

In `packages/opencode/src/project/project.ts`:

- Line 241 (deleted) — the worktree was being computed *before*
  `topLevel` was resolved and *before* `sandbox` was reassigned to
  `topLevel.text.trim()` at line 274 of the post-diff file. That early
  computation used the still-pre-`topLevel` value of `sandbox` for the
  `common === sandbox` comparison and used `common` (the bare repo
  path) as the worktree result for the bare-repo branch.
- Line 274 (added) — the same expression is now evaluated *after*
  `sandbox = resolveGitPath(sandbox, topLevel.text.trim())`, and the
  bare-repo branch returns `sandbox` (which is now the real
  `topLevel`/working tree path) instead of `common` (the bare
  directory). The non-bare branch still returns
  `pathSvc.dirname(common)` which is the worktree-of-`.git`
  convention and was already correct.

Net behavioral pivot: bare-repo worktrees now expose
`project.worktree === <checked-out worktree path>` while
`project.id` and the cache-location logic (still keyed off `common`)
are unchanged, so cache files continue to live under the bare repo
exactly as before. The fix splits the two concerns that were
incorrectly coupled.

## Test coverage

Three test updates at `packages/opencode/test/project/project.test.ts`
align with the behavioral pivot:

- `worktree from bare repo should use checked out worktree and cache in bare repo`
  (renamed from `…cache in bare repo, not parent`) at lines 522-545 —
  asserts `project.worktree).toBe(worktreePath)` (was `barePath`),
  adds `expect(project.sandboxes).not.toContain(worktreePath)` to pin
  that the worktree path isn't redundantly registered as a sandbox,
  and keeps the existing cache-location assertions
  (`barePath/opencode` exists, parent `.git/opencode` does not).
- Multi-worktree test at lines 555-578 — adds explicit
  `expect(projA.worktree).toBe(worktreeA)` and `…toBe(worktreeB)`
  assertions confirming that two separate worktrees of the same bare
  repo each get their own `worktree` value (previously both would
  have been `barePath`, indistinguishable).
- Subdirectory-of-worktree test at line 598 — flips the assertion
  from `barePath` to `worktreePath` to match the corrected behavior.

The cache-location assertions are deliberately preserved across all
three tests, which is the right way to demonstrate that this is a
narrow `worktree`-field fix that does not regress the
already-correct cache-key behavior.

## Why merge-as-is

- 4 line change with the right structural rationale: moving the
  `worktree` computation past the point where `sandbox` is reassigned
  to `topLevel`. Pre-existing `common === sandbox` short-circuit
  preserved verbatim, so the non-bare path is bit-identical.
- The `isBareRepo ? sandbox : pathSvc.dirname(common)` ternary is
  correct on all three axes — non-worktree (`common === sandbox`
  returns `sandbox` early), bare-repo worktree (returns
  `sandbox` = working tree path), regular-repo worktree (returns
  `dirname(common)` = `.git`'s parent = worktree root).
- Tests exercise the three real-world layouts (single worktree,
  multi worktree of same bare, subdir of worktree) and pin both the
  new behavior and the preserved cache-location invariant.
- Risk surface is contained to the bare-repo branch — users not on
  bare-repo layouts see zero change.

## Optional nits (not blocking)

- The renamed test name `worktree from bare repo should use checked
  out worktree and cache in bare repo` is accurate but long; a
  shorter `…uses checked out path, caches in bare` would scan
  better in test output. Pure cosmetic.
- The fix doesn't add a comment at line 274 explaining *why*
  computation moved past `topLevel` resolution; a one-line "must run
  after `sandbox = topLevel` so bare-repo worktree path is
  populated" would help the next reader avoid moving it back.
