# Review — google-gemini/gemini-cli #26401: fix(core): handle ENAMETOOLONG in robustRealpath

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26401
- **Head SHA:** `1406a5debd8c76d3b481d2d59eb6e2b1fc2b0737`
- **Author:** senutpal
- **Files:** 3 changed (+118 / −2)
  - `packages/core/src/utils/paths.ts` (+6/−2)
  - `packages/core/src/utils/paths.test.ts` (+50)
  - `packages/cli/src/ui/hooks/atCommandProcessor.test.ts` (+62)

## Verdict: `merge-after-nits`

## Rationale

Targeted fix for a real crash (issue #26368). When a user pasted a long blob containing an unescaped `@`, the `@`-tokenizer in `atCommandProcessor` routed the post-`@` substring through `robustRealpath`, which called `fs.realpathSync` and (on the fallback path) `fs.lstatSync`. On macOS those raise `ENAMETOOLONG` for over-long leaf segments, the error escaped `robustRealpath`, surfaced as an unhandled promise rejection inside the async `checkPermissions` flow, and crashed the CLI. The fix at `paths.ts:425-449` adds `ENAMETOOLONG` to both error-code allowlists (the outer `realpathSync` catch and the inner `lstatSync` catch), letting the function fall through to its parent-directory resolve path instead of throwing.

The two-character delta in production code is the right surgical change — `robustRealpath` already had the recovery shape, it just needed `ENAMETOOLONG` in the predicate. The fall-through behaviour for a single-segment over-long path is correct: `path.dirname(p) === p` returns the input unchanged via `path.resolve(longString)` (verified by the test at `paths.test.ts:597-607`), and downstream consumers (`validatePathAccess`, `fileExists`) will then treat it as "not a real path" and the at-command handler will skip permission entries.

Test coverage is thorough: three unit tests in `paths.test.ts:566-611` covering (a) the basic `ENAMETOOLONG` no-throw guarantee, (b) the resolve-and-return path, and (c) the fallback-into-`lstatSync` case where the symlink branch fires. The integration-level coverage in `atCommandProcessor.test.ts:1546-1604` verifies the actual user-facing scenario — a 5000-char `@`-token doesn't reject — which is the right guard against future regression at a different layer.

Two nits. First, the `mockConfig` in `atCommandProcessor.test.ts:1564-1574` casts via `as unknown as Config` and stubs only the four methods `checkPermissions` calls; this is fragile — if `categorizeAtCommands` adds a new `Config` accessor, the test will start throwing TypeError instead of cleanly failing the assertion. Consider extracting a shared mock-config factory. Second, the test description "5000 chars is well over macOS PATH_MAX (1024) and Linux PATH_MAX (4096)" is correct for `PATH_MAX`, but the leaf-segment limit (`NAME_MAX`) is what actually triggers `ENAMETOOLONG` here (255 on most filesystems). Minor wording, doesn't affect correctness.

## Banned-string check

Diff scanned; no banned tokens present.
