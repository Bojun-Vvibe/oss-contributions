# PR #26330 — fix(cli): ensure branch indicator updates in sub-directories and worktrees

- Repo: google-gemini/gemini-cli
- Head: `1f73e6664cfc4d9e86d7d8b3a63fd0764c691ef6`
- URL: https://github.com/google-gemini/gemini-cli/pull/26330
- Verdict: **merge-after-nits**

## What lands

Closes #19271. Pre-fix `useGitBranchName` assumed `.git/logs/HEAD` existed
relative to the cwd, which broke in two cases the bug report covers:
sub-directories (where `.git` is one or more levels up) and git worktrees
(where the `.git` *file* points to a separate worktree-specific gitdir under
`<repo>/.git/worktrees/<name>/HEAD`, not a `.git` directory at the cwd at
all).

Fix shape (test-side diff visible at `useGitBranchName.test.tsx`):

1. Resolve the actual git directory dynamically via `git rev-parse
   --absolute-git-dir` instead of joining cwd with `.git`.
2. Watch the resolved directory's `HEAD` file (not `logs/HEAD`) — `HEAD`
   reliably changes on branch checkout *and* on detached-HEAD commits, which
   is what the indicator needs to refresh on.

The test rewrite at `useGitBranchName.test.tsx:43-235` is the load-bearing
review evidence: the previous test mocked `GIT_LOGS_HEAD_PATH = path.join(CWD,
'.git', 'logs', 'HEAD')` but the new test correctly mocks
`GIT_HEAD_PATH = path.join(GIT_DIR, 'HEAD')` and the `watch` mock at
`:207-216` asserts `watchSpy` was called with `GIT_DIR` (the resolved
absolute-git-dir, not `cwd/.git`). The new `resolveInitialSpawns` helper at
`:96-118` correctly handles the new pair of `--abbrev-ref` plus
`--absolute-git-dir` spawns that fire concurrently from `setupWatcher` and
`fetchBranchName`, with `--short` handled for the detached-HEAD case.

## Nits

- The `resolveInitialSpawns` helper drains the `deferredSpawn` queue with a
  `while (deferredSpawn.length > 0)` loop and a `await new Promise((r) =>
  setTimeout(r, 0))` between iterations — works but is a code smell. If a
  fourth concurrent spawn type appears later, this helper silently drops it
  via the unhandled `else` branch. Add `else { throw new Error(\`unexpected
  spawn args: ${spawn.args}\`) }` so future contract changes fail loudly.
- The detached-HEAD path adds a `waitFor` polling loop at `:158-162` checking
  `if (!deferredSpawn.find((s) => s.args.includes('--short'))) throw ...` —
  this is a polling pattern around a queue that should ideally be event-driven.
  Acceptable for tests; `waitFor` already retries.
- The `watch` mock at `:207-216` casts via `as unknown as typeof fs.watch`
  which loses the type contract. Consider a typed `MockedFunction<typeof
  fs.watch>` from vitest helpers.
- No explicit test for the *worktree* case — only "sub-directory" is exercised
  via the `--absolute-git-dir` resolution. A test mocking `--absolute-git-dir`
  to return `/test/project/.git/worktrees/feature-x` (the worktree-specific
  gitdir shape) would lock the second half of the fix's stated scope. Pre-fix
  bug #19271 explicitly mentions worktrees in its repro.
- The behavior on `git rev-parse --absolute-git-dir` *failure* (e.g. cwd is
  not in any git repo) is not covered by a new test. The `error case` test
  only rejects `--abbrev-ref`. Add a test where `--absolute-git-dir` rejects.

## Why merge-after-nits not merge-as-is

The fix is correct and the test diff is well-shaped, but the missing worktree
test is the load-bearing one — it's literally half the bug's stated scope and
the diff's mock infrastructure is set up for it; one more `it()` block would
lock it. Combined with the `resolveInitialSpawns` silent-drop nit, this is
worth one review pass before merge.