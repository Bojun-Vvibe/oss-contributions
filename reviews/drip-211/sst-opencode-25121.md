# sst/opencode#25121 — fix(opencode): project .opencode/ overrides global ~/.opencode

- PR: https://github.com/sst/opencode/pull/25121
- Head SHA: `e94abf0faf681e0ea631084c05a83c3f60a5275b`
- Author: bainos (Jacopo Binosi)
- Files: `config/paths.ts` +5/−5; closes #19296, #21307

## Context

`ConfigPaths.directories()` returns the ordered list that
`unique(...)` later collapses; downstream config-merge treats the
*last* entry as the highest-precedence override. Pre-PR ordering
was: `[Global.Path.config, ...projectWalk(stop=worktree),
...homeWalk(stop=$HOME), ...]`. Because `unique()` keeps the first
occurrence and *drops the later duplicate*, the home `~/.opencode`
entry — emitted last — was never deduped against an earlier walk
hit, but more importantly the project walk results were sandwiched
between the global config and the home directory, so the home
walk was processed by config-merge after the project walk.
Net effect: a `model: X` in `~/.opencode/agent/build.md` always
beat `model: Y` in `<project>/.opencode/agent/build.md`, which is
the inverse of every documented precedence claim.

## Design

The fix at `config/paths.ts:26-43` swaps the two `afs.up()` spreads:

```
[Global.Path.config,
 ...homeWalk(start=$HOME, stop=$HOME),     // moved up
 ...(!OPENCODE_DISABLE_PROJECT_CONFIG
   ? projectWalk(stop=worktree) : []),    // moved down — now last
 ...(OPENCODE_CONFIG_DIR ? [...] : [])]
```

Now `unique()` sees `~/.opencode` early, then the project walk
(which by definition cannot include `~/.opencode` as an ancestor
unless the project lives directly under `$HOME`, in which case
the home entry wins the dedup — correct, because that single
directory *is* both the home and the project root). Project walk
results land last, so config-merge sees them as the highest-
precedence overrides.

The `start=$HOME, stop=$HOME` shape on the home walk is correct:
it produces exactly one entry (`~/.opencode`), no walk, no
ancestor traversal — which is the right shape for "global user
config, no upward inheritance".

## Risks / nits

- The 5-line diff has no test addition. The PR description's
  manual repro (`agent/build.md` with model X vs Y) is the right
  shape for a regression test: a 20-line `paths.test.ts` case
  setting both files in a tmpdir and asserting the project entry
  appears after the home entry would lock the precedence and
  catch any future re-ordering. Worth one round-trip with the
  maintainer to add it.
- The pre-PR comment-free swap leaves no breadcrumb explaining
  *why* the order matters. A one-line `// project walk last so
  unique() preserves project precedence over $HOME` above the
  closing bracket would prevent the next refactor from "tidying"
  the order again. (#19296 has been open long enough that this
  has clearly bitten people who didn't grep the issue tracker.)
- `OPENCODE_CONFIG_DIR` still lands last and beats the project
  walk, which matches the existing CLI-flag-wins-everything
  contract documented in #21307. Correctly preserved.

## Verdict

**`merge-after-nits`**

Real bug (silent global-overrides-project precedence inversion,
two open issues spanning months), minimum-blast-radius fix
(swap two spreads, no surface or schema change), correct
reasoning about `unique()` semantics. Nits are a regression
test plus a one-line precedence comment — both worth landing
in the same PR but not blockers.

## What I learned

`unique()` semantics ("keep first occurrence, drop later
duplicates") combined with config-merge semantics ("later
wins") creates a load-bearing ordering invariant that is
invisible at the call site. Any list-of-config-paths function
should ship a comment naming which end is highest-precedence,
because the two libraries that meet at this seam (dedup +
merge) have opposite first-vs-last conventions and the bug
will resurface as soon as someone tries to add a fourth source.
