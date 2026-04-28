# google-gemini/gemini-cli #26074 — fix(core): handle ENAMETOOLONG/ENOTDIR in robustRealpath

- PR: https://github.com/google-gemini/gemini-cli/pull/26074
- Head SHA: `d31b0158e731`
- Diff: 48+/14- across 2 files (`packages/core/src/utils/paths.test.ts`, `packages/core/src/utils/paths.ts`)
- Base: `main`
- Fixes: #26010

## Verdict: merge-as-is

## Rationale

- **Closes a real "pasted long string crashes the at-mention parser" bug.** Today, `robustRealpath` at `paths.ts:421-462` catches only `ENOENT` / `EISDIR` from `fs.realpathSync`, then re-throws everything else. The at-mention parser (`atCommandProcessor`) probes pasted strings as potential file paths, and a string longer than the OS's `PATH_MAX` (typically 4096 on Linux, 1024 on macOS) raises `ENAMETOOLONG`, which propagates up as an unhandled rejection. Similarly, an `@path/to/file.json/extra` mention surfaces `ENOTDIR` (intermediate file in path) and crashes the same caller. The PR adds a third catch arm at `:451-458` returning the input path unchanged for both error codes, with a load-bearing comment naming the at-mention probe pattern as the use case (`:452-456`).
- **Refactor-then-feature is the right shape.** The duplicated `e && typeof e === 'object' && 'code' in e && (e.code === 'X' || e.code === 'Y')` check at `:421-426` and `:436-444` is extracted into a typed helper `hasErrorCode(e: unknown, codes: readonly string[])` at `:415-421` and reused at three sites (`:432`, `:443`, `:451`). The extraction is mechanically correct (the helper preserves the same semantics: `false` for non-object / no `code` / non-string `code` / code-not-in-list) and removes the diff hazard of having three near-identical predicates that could drift. Net 6 lines saved + a clearer call-site shape.
- **The new `ENAMETOOLONG`/`ENOTDIR` arm at `:451-458` returns `p` unchanged** — *not* the parent path, *not* `null`, *not* a thrown error. That's the right semantic: the input *isn't* a real path, but the caller's contract (as documented in the inline comment) is "let downstream `fileExists` / `fs.stat` reject it normally". Any other choice (return `null`, throw a typed error, return the parent) would force every caller to add a new error-handling branch; returning the input means existing callers' downstream `fs.stat(p)` / `fileExists(p)` calls produce the *same* `ENOENT` / `ENOTDIR` they already know how to handle.
- **Test coverage is precise.** `paths.test.ts:564-576` mocks `fs.realpathSync` once with an `ENAMETOOLONG` error and asserts `resolveToRealPath(longInput)` returns `longInput` unchanged. `:578-590` does the symmetric `ENOTDIR` case. Both tests use `vi.spyOn(...).mockImplementationOnce(...)` so the mock is scoped to the single call, preventing cross-test pollution. The error construction at `:567-570` and `:582-585` uses the canonical `Error & { code }` cast pattern that mirrors how Node.js raises `NodeJS.ErrnoException` instances, so the test is exercising the real shape.
- **The comment block at `paths.ts:451-456` is unusually good operator-UX.** It names the issue (`#26010`), the trigger (the at-mention probe in `atCommandProcessor`), and the *reason* the function returns the input unchanged ("lets downstream fileExists / fs.stat calls reject it normally instead of surfacing an unhandled rejection"). That kind of "this branch is correct because the caller relies on a different layer to reject" comment is exactly what prevents a future contributor from "fixing" the early-return into a thrown error.

## Nits / follow-ups

- **The `hasErrorCode` helper at `:415-421` could move to a shared module** if other path/fs utilities in the codebase repeat the same pattern. A quick `rg "code === 'ENOENT'" packages/core` would say whether the dedup opportunity extends beyond this file. Optional.
- **No test for `robustRealpath` being called recursively** with `ENAMETOOLONG` on the parent path resolution at `:455` (`return path.join(robustRealpath(parent, visited), path.basename(p))`). If the parent path is *also* `ENAMETOOLONG`-shaped (e.g. all 4096 chars are slashes), the recursion would re-trigger the new branch and short-circuit correctly, but a defensive test would lock that.
- **`readonly` on the `codes` parameter at `:415` is the right type-discipline choice** — it lets callers pass `['ENOENT', 'EISDIR']` (a non-readonly tuple) without TypeScript widening, but blocks the helper from ever mutating the input. Good idiom.
- **The inline comment at `:451-456` mentions "the at-mention path parsing in atCommandProcessor"** — worth a `// see also: <file>:<line>` pointer so future readers can navigate to the exact caller. Optional.

## What I learned

The "catch this specific error code, ignore everything else" pattern in fs-error handling is a well-known fail-loud-vs-fail-quiet decision. The bug here is the canonical "the original code only thought about the two most common errors" trap — `ENAMETOOLONG` and `ENOTDIR` are both rare in normal use but extremely common in adversarial / fuzzy input (pasted blobs, malformed @-mentions). The fix is small but the lesson is twofold: (a) extract the error-code check into a helper before adding new codes — it's the only way to keep the predicate consistent across multiple call sites, (b) when the caller's contract is "let me probe", the right return for "this isn't a real path" is the input unchanged, not a thrown error — because the caller is *already* prepared to handle the path being unresolvable downstream.
