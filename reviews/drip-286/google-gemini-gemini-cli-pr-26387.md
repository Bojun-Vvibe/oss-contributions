---
repo: google-gemini/gemini-cli
pr: 26387
head_sha: 25518e8e8e395d545cc6d33fc5fb3d95a7ad8d2a
title: "fix(core): implement system ripgrep fallback when bundled binary is missing"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: google-gemini/gemini-cli#26387 — `fix(core): implement system ripgrep fallback when bundled binary is missing`

**Head SHA:** `25518e8e8e395d545cc6d33fc5fb3d95a7ad8d2a`
**Stat:** +6 / −0 across 1 file. Author: chaitanyabisht. Fixes #26193.

## What it changes

In `packages/core/src/tools/ripGrep.ts`, the existing `getRipgrepPath()`
function previously had two resolution steps (per the surrounding
context: bundled `vendor` binary, then a documented step 2 lookup).
Step 3 — system PATH — was missing, so users on NPM-installed
distributions where #25841 stripped the architecture-specific bundled
binaries were falling all the way through to the JS-based GrepTool
even when they had `rg` installed system-wide.

The fix adds:

```ts
// 3. Fallback to system PATH
if (isBinaryAvailable('rg')) {
  return 'rg';
}
```

at line 65–68, plus the import of `isBinaryAvailable` from
`../utils/binaryCheck.js` at line 41.

## Assessment

- This is exactly the missing fallback path described in #26193. The
  optimization in #25841 (drop arch-specific binaries from the NPM
  tarball to slash package size) was the right move; this PR closes
  the regression that optimization opened for users with
  system-installed `rg`.
- Numbered comment (`// 3. Fallback to system PATH`) follows the
  existing convention in the function — keeps the resolution-order
  documentation visible at the call site. Important because the next
  person to add a step needs to know it's an ordered list, not an
  arbitrary set.
- Returning the bare string `'rg'` (rather than an absolute resolved
  path) means the spawn site relies on `child_process` PATH resolution
  to find the binary at exec time. That's standard Node behavior and
  matches what the warning message ("system `rg`") implies — but
  worth checking that the spawn path doesn't anywhere assume an
  absolute path (e.g. for sandboxing/permission checks). If it does,
  this should resolve via `which rg` and return the absolute path.
- `isBinaryAvailable('rg')` is presumably a `which`-style PATH probe;
  the implementation lives in `utils/binaryCheck.js` (not in this
  diff). Worth confirming it's synchronous and doesn't itself spawn a
  process — `getRipgrepPath` is async (returns `Promise<string |
  null>`) so a sync check is fine, but `isBinaryAvailable` being sync
  + called on every grep is the desired shape.
- No new test in this PR. PR description claims local verification +
  `--debug` log inspection + `npm run preflight`, but a unit test that
  asserts "PATH-only `rg` is detected when bundled is missing" would
  be valuable since #25841 + this PR together change the binary-
  resolution behavior for the most common install path.

## Nits

- (Non-blocking) Add a unit test that mocks `isBinaryAvailable` and
  the bundled-path check to assert all three resolution branches.
  6-line behavior change, ~30-line test — high signal-to-noise.
- (Non-blocking) Consider returning the absolute path resolved from
  PATH (e.g. via `which`/`isBinaryAvailable` returning the path it
  found) rather than the bare `'rg'`. Defends against PATH changes
  between binary resolution and binary spawn, and produces clearer
  debug logs.
- (Non-blocking) When the system PATH fallback fires, consider a
  one-time debug log ("using system rg at X") so users hitting this
  path can see in `--debug` output why the previously-warned
  "Ripgrep is not available" message stopped appearing.

## Verdict

**`merge-after-nits`** — minimal correct fix to a real regression.
The behavior change is right; the polish nits (unit test, absolute
path) are worth landing as follow-ups but shouldn't block this 6-line
fix.
