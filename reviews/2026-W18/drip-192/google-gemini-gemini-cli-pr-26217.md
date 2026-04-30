# google-gemini/gemini-cli #26217 — test(cleanup): fix temporary directory leaks in test suites

- **URL:** https://github.com/google-gemini/gemini-cli/pull/26217
- **Head SHA:** `076e593dbce3a478e152185ce1959888bbad678b`
- **Files:** `packages/cli/src/commands/extensions/configure.test.ts` (+15/-3), `packages/cli/src/config/extension-manager-scope.test.ts` (+13/-2), `packages/core/src/tools/mcp-client.test.ts` (+53/-4)
- **Verdict:** `merge-after-nits`

## What changed

Adds explicit tmpdir cleanup in `afterEach` hooks across three test suites that previously created tempdirs in `beforeEach` but relied on OS-level cleanup. Two real bugs fixed:

1. **`configure.test.ts:89`** previously called `fs.mkdtempSync('gemini-cli-test-workspace')` with a *relative* template — so on Linux/macOS the tempdir was created in the test's CWD (often the repo root), not in `os.tmpdir()`. The fix at `:89-91`:
   ```ts
   tempWorkspaceDir = fs.mkdtempSync(
     path.join(os.tmpdir(), 'gemini-cli-test-workspace-'),
   );
   ```
   pushes it to the OS tempdir + adds the canonical trailing `-` separator so `mkdtempSync`'s 6-char suffix doesn't smear into the prefix.

2. **`mcp-client.test.ts`**: four separate `afterEach` blocks (`McpClient` describe at `:108`, `connectToMcpServer with OAuth` at `:2421`, `connectToMcpServer - HTTP→SSE fallback` at `:2640`, `connectToMcpServer - OAuth with transport fallback` at `:2815`) all gain the same cleanup pattern: `fs.rmSync(testWorkspace, { recursive: true, force: true })` wrapped in try/catch + a Windows-specific `setTimeout(100ms)` before delete to let the OS release file handles + `workspaceContext = null as unknown as WorkspaceContext` to drop GC-roots. Same pattern in `extension-manager-scope.test.ts:90-101` for two dirs (`currentTempHome`, `tempWorkspace`).

## Why it's right

- **The `mkdtempSync('gemini-cli-test-workspace')` bug is real.** Node's `fs.mkdtempSync(prefix)` interprets the prefix as a *path* — a relative prefix means "create in CWD". On a CI worker that runs tests from the repo root, this litters the repo with `gemini-cli-test-workspace*` directories on every test run, which (a) shows up as untracked files in `git status` for the next test that does a `git`-aware operation, and (b) eventually fills the disk on a long-running CI agent. Moving to `path.join(os.tmpdir(), 'gemini-cli-test-workspace-')` is the canonical fix and matches how `mkdtempSync` is used everywhere else in this repo.
- **Trailing `-` on the prefix matters.** `mkdtempSync('foo')` produces `fooXXXXXX`, `mkdtempSync('foo-')` produces `foo-XXXXXX`. The latter is greppable / globbable as a distinct family; the former smears into any dir starting with `foo`. Small but right-thinking detail.
- **`fs.rmSync(dir, { recursive: true, force: true })` is the post-Node-14.14 canonical pattern.** Replaces the deprecated `rmdirSync({ recursive: true })`. `force: true` swallows ENOENT so tests that already cleaned up don't error in `afterEach`.
- **Windows 100ms wait is empirically necessary.** Windows holds file handles open longer than POSIX (especially with antivirus / Defender scanning new files). Tests that write a file then immediately `rmSync` the parent dir get EBUSY/EPERM races. The 100ms wait is the standard mitigation in Node test suites; the conditional `if (process.platform === 'win32')` keeps Linux/macOS fast.
- **Try/catch with `// ignore` is correct here.** `afterEach` cleanup *must not* throw — if it does, the test that owned the cleanup gets reported as failed even though the test itself succeeded, producing red CI from green tests. A swallowed cleanup error is a leaked tempdir, not a wrong result; failing the test for it would invert the priority. The comment `// ignore` is honest — it makes the swallow intentional rather than accidental.
- **`workspaceContext = null as unknown as WorkspaceContext`** at `mcp-client.test.ts:124` and three other sites. This is the GC-rooting fix: tests that hold a `WorkspaceContext` referencing the deleted dir prevent V8 from reclaiming objects that may transitively hold open file watchers. The cast is a TypeScript necessity (the field type is non-nullable); the runtime behavior is what matters.

## Nits

1. **Five copies of the same cleanup block.** All four `afterEach` blocks in `mcp-client.test.ts` plus the one in `extension-manager-scope.test.ts` are *literally* identical: same try/catch, same Windows wait, same null assignment, same `// ignore`. A shared `await safeRemoveTempDir(dir)` helper in a test utility would collapse five copies to one and would let a future Windows-handle-leak fix land in one place. Worth a follow-up.

2. **`vi.useRealTimers()` placement is now inconsistent.** The new `afterEach` in `mcp-client.test.ts:108-127` puts `vi.useRealTimers()` *first*, then cleanup, then `vi.restoreAllMocks()`. But the other three new `afterEach` blocks (`:2421`, `:2640`, `:2815`) put `vi.useRealTimers()` first too, while the original code at `:108` had `vi.restoreAllMocks()` *then* `vi.useRealTimers()`. The new ordering is correct (real timers before cleanup so any setTimeout-based async fires deterministically) but the comment doesn't say so. A one-line `// real timers first so the Windows wait actually elapses` would prevent a future patch reverting the order.

3. **Windows wait is hard-coded 100ms.** Empirically fine, but on a slow Windows CI agent under load (with Defender scanning) 100ms may be insufficient; on a fast one it's wasteful. A configurable `WINDOWS_RM_DELAY_MS = parseInt(process.env.GEMINI_TEST_WIN_RM_DELAY ?? '100', 10)` would give a flaky-test escape hatch without a recompile. Low priority.

4. **`tempWorkspaceDir && fs.existsSync(tempWorkspaceDir)`** at `configure.test.ts:91`. The `&&` short-circuits if `tempWorkspaceDir` is falsy (e.g. `beforeEach` failed before assigning), which is right. But the variable is declared without an explicit `string | undefined` type — a future patch that initializes it to `''` (truthy-empty for path semantics but truthy-true for JS) would `existsSync('')` which on some Node versions returns `true` for the CWD. Adding `let tempWorkspaceDir: string | undefined;` at the declaration would prevent that footgun.

5. **`mcp-client.test.ts` `afterEach` count balloons.** Four near-identical `afterEach` blocks across four describe blocks suggests the per-describe `testWorkspace` variable is actually shared state by another name. If all four describes can be folded into a `describe.each`-style parametrization or share a top-level `beforeEach`/`afterEach`, the file would shrink and the cleanup logic would centralize. Out of scope for this PR but worth a tracking note.

## Risk

Very low. Test-only change; no production code path touched. Failure modes: (a) cleanup `try/catch` swallows a cleanup error so a tempdir leaks — same as today's behavior, still better than before because at least the *attempt* is made. (b) Windows 100ms wait adds ~100ms × 4 describes × N test invocations to total Windows CI runtime — measurable but not significant on a normal suite. (c) The relative→absolute `mkdtempSync` change in `configure.test.ts` could affect any test that *depended* on the workspace being in CWD (none should, but worth a CI watch on first merge).
