# google-gemini/gemini-cli PR #26387 — fix(core): implement system ripgrep fallback when bundled binary is missing

- Author: chaitanyabisht
- Head SHA: `25518e8e8e395d545cc6d33fc5fb3d95a7ad8d2a`
- Diff: +6 / -0 across 1 file
- File: `packages/core/src/tools/ripGrep.ts`

## Observations

1. **The entire diff is `ripGrep.ts:41` (import) and `:65-68` (fallback)**: when the bundled `rg` binary path-resolution returns nothing, the function now checks `isBinaryAvailable('rg')` and returns the bare string `'rg'` (relying on the spawn layer to PATH-resolve it). This is the correct minimal fix for environments where the bundled binary failed to ship (Linux distro packagers stripping it, security tooling quarantining it, npm install glitches).
2. **Behavior change is opt-in by environment**: users without a system `rg` see the same `null` return as before, so callers that handle "ripgrep unavailable" gracefully are unaffected. Good blast-radius control.
3. **No tests added**. For a 6-line PR this is borderline acceptable, but a single unit test mocking `isBinaryAvailable` to return `true` and asserting the function returns `'rg'` would prevent a future refactor from silently dropping the fallback. Strongly recommended.
4. **`isBinaryAvailable` provenance**: imported from `'../utils/binaryCheck.js'`. I did not see the implementation in this diff — assuming it does a `which`/`where` shell-out or scans PATH. Verify it's synchronous (this function is `async` but the new call isn't `await`ed) — `getRipgrepPath` is `async`, but `isBinaryAvailable('rg')` is used without `await`, implying a synchronous API. If it's actually async, this code returns `'rg'` for every truthy Promise reference, which would be a serious bug. Quick check needed.
5. **Returning the bare string `'rg'` vs an absolute path**: downstream code that previously could rely on `getRipgrepPath()` returning either an absolute path or `null` now must accept a third state — a relative command name to be PATH-resolved at spawn time. Confirm callers don't, e.g., `fs.access(path)` on the result before spawning, which would fail on the bare `'rg'`.
6. **Comment numbering**: the new block is "// 3. Fallback to system PATH" — implies the existing logic is split into 1 and 2. Worth verifying the surrounding code actually has those numbered comments; if not, add them for symmetry, otherwise the lone `3.` is jarring.

## Verdict: `merge-after-nits`

Tight, useful fix that solves a real packaging gotcha. Two follow-ups before merge: (a) **critical** — confirm `isBinaryAvailable` is synchronous (the unawaited call is the only correctness risk in this diff); (b) add a unit test pinning the fallback behavior. The bare-string return convention should also be sanity-checked against downstream `fs.access`/`stat` calls.
