---
repo: QwenLM/qwen-code
pr: 3784
head_sha: 69d71a69730e5840f9e70a74a5c67e582b159a2f
title: "fix(monitor): correct Windows taskkill spawn assertion to include stdio option"
verdict: merge-as-is
reviewed_at: 2026-05-03
---

# Review: QwenLM/qwen-code#3784 — `fix(monitor): correct Windows taskkill spawn assertion to include stdio option`

**Head SHA:** `69d71a69730e5840f9e70a74a5c67e582b159a2f`
**Stat:** +15 / −18 across 1 file (`packages/core/src/tools/monitor.test.ts`).
State: MERGED. Approved by doudouOUC.

Reviewing post-merge for the drip log; comments are advisory.

## What it changes

Three `toHaveBeenCalledWith('taskkill', [...])` assertions in the
`MonitorTool` test file's Windows branches (lines 712, 747, 783)
asserted only two arguments to `mockSpawn`, while the implementation
in `monitor.ts`'s `killChildProcessGroup` calls `spawn('taskkill',
[...], { stdio: 'ignore' })` — three arguments. Vitest's
`toHaveBeenCalledWith` is exact-match, so all Windows CI jobs were
red on this assertion mismatch.

The fix updates each of the three assertions to include the
`{ stdio: 'ignore' }` third argument, e.g. at line 713–717:

```ts
expect(mockSpawn).toHaveBeenCalledWith(
  'taskkill',
  ['/pid', '12345', '/f', '/t'],
  { stdio: 'ignore' },
);
```

## Assessment

- This is a test-only fix to align an assertion with an existing
  production signature. Zero production-code risk.
- Three call sites updated, all with identical shape — symmetric and
  easy to audit at a glance.
- The fix correctly matches the implementation contract (`spawn` with
  the `stdio: 'ignore'` option) rather than the inverse (loosening
  the test or relaxing the production code). Right direction.
- The author validated locally on Linux via `npx vitest run
  src/tools/monitor.test.ts` (64 tests pass). The actual win32 branch
  still requires Windows CI to exercise — that's the whole point of
  this fix, so the PR is self-validating: if the next Windows CI run
  goes green, this worked.
- The Linux/macOS branches in the same `if (process.platform ===
  'win32')` blocks are untouched, and they assert against `killSpy`
  rather than `mockSpawn` — different code paths, correctly left
  alone.

## Nits

(Post-merge — advisory only.)

- (Non-blocking) When asserting against a `spawn` mock with options,
  consider extracting a small helper like
  `expectTaskkillSpawn(mockSpawn, '12345')` so the three identical
  shapes don't drift if the implementation later adds another option
  (e.g. `windowsHide: true`). With three call sites the duplication
  is fine; if a fourth ever lands, the helper pays off.
- (Non-blocking) If the goal is just "verify taskkill was called with
  these args, don't care about future option additions",
  `expect.objectContaining({ stdio: 'ignore' })` would be more
  resilient to future option additions. But strict-match is also a
  valid policy for spawn args — both are defensible.

## Verdict

**`merge-as-is`** (already merged) — minimal, correct test alignment
fix. Right scope, right direction, right validation.
