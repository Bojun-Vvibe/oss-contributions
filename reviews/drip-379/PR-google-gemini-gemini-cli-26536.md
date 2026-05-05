# google-gemini/gemini-cli #26536 — feat(core): add system-wide fallback for ripgrep detection

- URL: https://github.com/google-gemini/gemini-cli/pull/26536
- Head SHA: `dbd30cab4789da90d6fccb35b0347b41a5e197bd`
- Author: alexandrevarga
- Size: +22 / -2 across 2 files

## What it does

Adds a PATH-based fallback to `getRipgrepPath()` at `packages/core/src/tools/ripGrep.ts:64-72`: after the existing bundled-binary check (`fileExists` checks at `:51-63`, above the diff window), the function now tries `await spawnAsync('rg', ['--version'])` and returns the literal string `'rg'` on success, falling back to `null` only if both bundled and PATH lookups fail. Updates the existing "should throw an error if ripgrep is not available" test to also stub `mockSpawn` to emit a synchronous `'spawn rg ENOENT'` error so the fallback path returns null and the original error assertion still holds.

## Observations

- **Returning the bare string `'rg'` (vs an absolute path) means downstream `spawn(getRipgrepPath(), [...])` will re-resolve via PATH on every invocation.** If the user's PATH changes mid-session (unlikely but possible in long-lived TUI sessions where shell-level PATH is dynamic) or if a malicious `rg` is later inserted into a higher-precedence PATH directory (less unlikely), the resolved binary differs from the one validated at startup. A `which`/`where`-style absolute-path resolution at fallback time would close that TOCTOU window. Probably fine in practice given typical CLI lifetimes, but worth noting.
- **The `rg --version` smoke test at `:67` has unbounded latency.** `spawnAsync` likely has its own timeout, but if not, an `rg` binary that hangs on `--version` (rare but possible with a wrapper script masquerading as `rg`) would block tool initialization. A short timeout (e.g. 2s) on this probe would be defensive.
- **Comments in Portuguese in the test (`:9` "Simula que os arquivos internos não existem", `:13` "Simula que a execução do 'rg --version' falha síncronamente", `:18` "Emitimos o erro imediatamente").** The rest of the codebase uses English comments — these should be translated for consistency before merge. Minor.
- **No new positive-path test.** There's a regression test for the "neither bundled nor PATH has rg" case but no test asserting that when bundled fileExists returns false but PATH lookup succeeds, the function returns `'rg'` rather than `null`. A second `it()` covering the new positive path would make the new behavior pinned.
- The `// eslint-disable-next-line @typescript-eslint/no-explicit-any` at `:14` matches the surrounding mock-construction style; not a new pattern.
- The `vi.mocked(fileExists).mockResolvedValue(true)` restoration at `:30` is correctly preserved (the mock-restore comment was deleted but the restoration call itself remains, since the test runs serially and other tests rely on the restored default).

## Verdict: merge-after-nits

Useful UX fix — users with `rg` already on their PATH (e.g. via `brew install ripgrep`) shouldn't have to ship a bundled binary. Three low-cost nits before merge: translate the Portuguese test comments to English, add a positive-path test asserting `'rg'` is returned when only PATH succeeds, and consider resolving to an absolute path (or at minimum noting in a code comment that `'rg'` re-resolves per-spawn). The TOCTOU concern is theoretical; the comment-language and missing-positive-test issues are concrete.