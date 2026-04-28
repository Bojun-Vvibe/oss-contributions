# sst/opencode #24692 — fix(opencode): use directory as worktree for non-git projects

- URL: https://github.com/sst/opencode/pull/24692
- Head SHA: `6da33466f784e02da1b7d8d438b83cbb8addc4c1`
- Verdict: **merge-as-is**

## Review

- Two-line fix at `packages/opencode/src/project/project.ts:199-200` swaps the previous `worktree: "/"` / `sandbox: "/"` defaults to the actual `directory` resolved one stack frame up. The `"/"` defaults were a real footgun on non-git projects: any tool whose access list checks `worktree` (file read/write gates, sandbox bind mounts, `pwd`-relative shell exec) would have happily allowed reads anywhere on the filesystem, treating the entire host as in-scope.
- Placement is right — inside the `if (!dotgit)` branch, after the discovery has already settled on `directory` as the canonical project root, before the `ProjectID.global` return. Nothing else in the branch needed to change because `fakeVcs` is unaffected by the `worktree` value.
- The change is paired with `ProjectID.global` rather than minting a new project ID; defensible because non-git projects are still "global" from the persistence layer's perspective (no per-project state file). Worth confirming in CI that two non-git projects in different directories don't collide on the global ID for any cached worktree-keyed lookups, but at the call-site level `worktree` itself is now distinct so most cache keys that include it are safe.
- No test added. For a 2-line correctness fix in a code path that's previously been quietly wrong (the regression would be silent — wider-than-intended sandbox), a regression test asserting `worktree === directory` for the no-`.git` branch would be cheap insurance, but the change is small and obviously correct enough that gating on a test isn't worth blocking.
