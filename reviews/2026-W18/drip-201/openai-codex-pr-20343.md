# openai/codex #20343 — ci: increase Windows release workflow timeouts

- **Author:** bolinfest (Michael Bolin)
- **SHA:** `206eed7`
- **State:** MERGED
- **Size:** +6 / -5 across 2 files (`.github/workflows/rust-release-windows.yml`,
  `.github/workflows/rust-release.yml`)
- **Verdict:** `merge-as-is`

## Summary

Lifts the per-job `timeout-minutes` from 60 → 90 on two Windows release jobs in
`.github/workflows/rust-release-windows.yml:27,142` (`build-windows-binaries` and the
downstream signing/aggregation job at `:140`), and rewrites the explanatory comment
in `.github/workflows/rust-release.yml:50-53` to drop the now-stale "ideally 60
minutes" aspiration. The top-level `rust-release.yml` job was already at 90 minutes;
this brings the Windows-specific workflow file into alignment.

## Reasoning

The fat-LTO mainline release builds on Windows runners do legitimately push past 60
minutes (the comment correctly attributes this to fat-LTO build cost), and a release
that times out at 60 minutes wastes the entire job's worth of compute and operator
attention to restart. Aligning both workflow files to the same 90-minute headroom
matches the existing top-level policy and removes a source of "why did Windows time
out but Linux didn't" surprise. The diff touches only timeout values and a comment
— zero behavioral surface beyond "release jobs may now run up to 30 minutes longer
before being killed." The companion comment cleanup at `rust-release.yml:50-53`
collapses the two-paragraph "ideally this would be 60 minutes" narrative into a
single sentence, which matches what the file actually does.
