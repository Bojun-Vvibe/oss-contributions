# Review: google-gemini/gemini-cli #26401 — fix(core): handle ENAMETOOLONG in robustRealpath

- **Repo**: google-gemini/gemini-cli
- **PR**: #26401
- **Head SHA**: `1406a5debd8c76d3b481d2d59eb6e2b1fc2b0737`
- **Author**: senutpal

## What it does

Stops `robustRealpath` from leaking `ENAMETOOLONG` as an unhandled
promise rejection when a user pastes a long blob containing an
unescaped `@`. The `@` triggers the path-resolution code path, which
then calls `fs.realpathSync` on the over-long string and crashes the
CLI. Issue #26368.

## Diff notes

- `packages/core/src/utils/paths.ts:425-449` — adds `ENAMETOOLONG`
  to the two error-code allowlists in `robustRealpath`. The first
  guard at line 425 lets the function fall through to `fs.lstatSync`
  on long paths; the second at line 437 (the lstat fallback)
  swallows `ENAMETOOLONG` from `lstatSync` itself, since macOS
  raises ENAMETOOLONG on the leaf-name lstat call rather than on
  realpath. Both branches are needed because the failing syscall
  differs by OS.
- The new comment at `paths.ts:439-440` explicitly cites issue
  #26368 — good. Future readers will understand why the error is
  swallowed instead of propagated.
- Three new tests in `paths.test.ts` (per the diff): one for
  `realpathSync` raising ENAMETOOLONG, one for the same with the
  resolved-path assertion, and one for `lstatSync` raising
  ENAMETOOLONG inside the symlink fallback. The third test is the
  one that actually proves the second code-path change works on
  macOS — without it, the change at line 437 would have no test.

## Concerns

1. The fix returns the input path unchanged when ENAMETOOLONG
   occurs. That's the right move (the caller — `validatePathAccess`,
   `fileExists` — will then treat it as "not a real path"), but no
   warning is logged. If a user genuinely tried to reference a long
   file (e.g. a base64 blob saved with that name), they'll get a
   silent fallthrough. Non-blocking — pasting an over-long string
   is the dominant case and a log line would be noisy.
2. The recovery is local to `robustRealpath`, but the same pattern
   (a bare `fs.realpathSync(longString)`) likely exists in other
   utilities. Worth a follow-up grep for `realpathSync\(` to confirm
   this is the only ingress point. The PR doesn't address that and
   probably shouldn't — scope-discipline is fine.
3. The new tests use `vi.spyOn(fs, 'realpathSync').mockImplementationOnce`
   for two cases and `mockImplementation` (no `Once`) for the third.
   The third test depends on the `mockImplementation` not bleeding
   into subsequent tests in the file — make sure there's an
   `afterEach(() => vi.restoreAllMocks())` in the surrounding
   `describe` block, or convert it to `mockImplementationOnce` for
   each call. Otherwise this could destabilize the suite later.

## Verdict

merge-after-nits
