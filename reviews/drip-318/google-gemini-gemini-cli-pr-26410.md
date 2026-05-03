# Review — google-gemini/gemini-cli#26410 — fix(cli): use os.homedir() for home directory warning check

- PR: https://github.com/google-gemini/gemini-cli/pull/26410
- Head: `28580cf12c0afe6c972f2ca0cdb07efaf0448dd0`
- Verdict: **request-changes**

## What the change does

In `packages/cli/src/utils/userStartupWarnings.ts` switches the home
directory comparison from the re-exported `homedir` (from
`@google/gemini-cli-core`) to `node:os`'s `homedir` directly, removes the
core re-export from the import list, and adds tests for symlinked-home and
`GEMINI_CLI_HOME`-override scenarios in `userStartupWarnings.test.ts`.

## Concerns (blocking)

1. **Broken module path in test mock.** Line 32 of the test:
   `await importOriginal<typeof import('@google-gemini-cli-core')>();`
   The dash-form package name `@google-gemini-cli-core` does not exist;
   the real package is `@google/gemini-cli-core` (with a slash). Either
   the original was correct and the PR introduced a regression, or this
   typo will silently make `importOriginal` return whatever the mock
   resolves and the rest of the spread becomes empty — both cases break
   the test for the wrong reason.
2. **Dropped mock value.** The previous mock exported
   `homedir: () => os.homedir()`, which other code paths in
   `userStartupWarnings` (or its callees inside core) may consume. By
   deleting the export from the mock without confirming there are no
   other call sites using the core re-export, a test could pass against a
   stale module shape. Search consumers of
   `import { homedir } from '@google/gemini-cli-core'` before merge.
3. **Behavior question.** The bug fix rationale is "use Node's `os.homedir`
   so symlinked $HOME is handled by the surrounding `fs.realpath`." That's
   correct, but the new tests demonstrate that `vi.mocked(os.homedir)`
   still controls the result — i.e. the production path now relies on
   `node:os` directly while tests stub `os.homedir`. Make sure no other
   callsite still imports `homedir` from core and expects a wrapped value
   (auth helpers, MCP config resolvers).
4. **Test cleanup placement.** The new `afterEach` is declared at the end
   of the `describe('home directory check', ...)` block but cleans up
   `testRootDir` — confirm `testRootDir` is scoped to that describe (it
   should be in `beforeEach`); otherwise root-dir-check tests below also
   inherit the cleanup.

Fix the package-name typo (#1) before anything else.
