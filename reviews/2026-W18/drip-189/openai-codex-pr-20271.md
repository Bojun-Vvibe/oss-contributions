# openai/codex#20271 — chore: increase release build timeout from 60 min to 90

- PR: https://github.com/openai/codex/pull/20271
- HEAD: `8582962f74289e436288f84832c5080b4b4e2644`
- Author: bolinfest
- Files changed: 1 (+4 / -1) — `.github/workflows/rust-release.yml`

## Summary

Bumps the per-job `timeout-minutes` on the release build matrix from 60 to
90 with an inline comment naming Windows builds as the slow path and the
pain of restarting a release build as the motivation. Pure release-tooling
ops change; no source-of-truth implications.

## Cited hunks

- `.github/workflows/rust-release.yml:52-55` — adds three comment lines
  ("Codex binaries can take a long time to build, particularly on
  Windows. Ideally, this would be 60 minutes, but let's add some headroom
  because having to restart a release build due to a timeout is a major
  pain.") and changes `timeout-minutes: 60` → `timeout-minutes: 90`.

## Risks

- 90 minutes still leaves a ceiling. If Windows builds are creeping toward
  60 minutes, the trajectory says they'll hit 90 in another ~6-12 months
  and someone will be doing this PR again. Worth a follow-up issue to
  track the actual P95 release build time per target so the next bump
  is informed by data, not pain.
- The `timeout-minutes: 90` is per-job, so it applies to every matrix
  cell (every runner × target × bundle combination). Fast macOS/Linux
  cells now silently allow themselves to hang for 90 minutes too, which
  is a regression on the "fail-fast on stuck job" property. A
  per-target timeout (Windows: 90, macOS/Linux: 60) via
  `${{ matrix.timeout || 60 }}` would keep the discipline on the cells
  that don't need the extra headroom.

## Verdict

**merge-as-is**

## Recommendation

Land it — release-build timeouts are a "fix the pain right now, optimise
the discipline later" class of problem and a 30-minute bump is the right
proportional response. Follow-ups: log per-target build times to a
metrics surface that can drive the next bump decision, and consider a
matrix-aware timeout if non-Windows cells start drifting upward too.
