---
pr: 19867
repo: openai/codex
sha: 0f697bc4c4bc190d248e35344cba56073150b532
verdict: merge-as-is
date: 2026-04-28
---

# openai/codex #19867 — Ignore legacy Windows cmd sandbox test in CI

- **Author**: evawong-oai
- **Head SHA**: `0f697bc4c4bc190d248e35344cba56073150b532`
- **Size**: 1 line added, 0 deleted (`codex-rs/windows-sandbox-rs/src/unified_exec/tests.rs`).

## Scope

One-line `#[ignore = "TODO: legacy non-tty cmd.exe fails with ERROR_DIRECTORY
in CI"]` attribute added to `legacy_non_tty_cmd_emits_output` so the legacy
cmd.exe sandbox path stops failing in hosted Windows CI. The PR body
correctly identifies the trigger: the parallel preserved-path policy stack
invalidates this test target through its protocol dependency, the test stops
coming from the build cache, and exposes a previously-cached `CreateProcessAsUserW`
error 267 (`ERROR_DIRECTORY`) on hosted runners. Adjacent legacy cmd.exe
tests are already `#[ignore]`-marked for the same class of CI process-launch
instability — this is a small consistency patch.

## Specific findings

- `codex-rs/windows-sandbox-rs/src/unified_exec/tests.rs:139` — single line
  added between `#[test]` and `fn legacy_non_tty_cmd_emits_output()`:
  ```rust
  #[ignore = "TODO: legacy non-tty cmd.exe fails with ERROR_DIRECTORY in CI"]
  ```
  Correct shape: uses the `#[ignore = "..."]` form with an embedded reason
  (not `#[ignore]` bare), so `cargo test -- --ignored` still runs it locally
  and the reason surfaces in test output. The "TODO:" prefix flags it for
  future cleanup.
- The `legacy_process_test_guard()` at the top of the test (now still
  reachable via `--ignored`) confirms the test is part of the legacy
  cmd.exe surface that already coordinates serialized access — so when
  someone *does* unblock this in the future, the guard still does its job.
- The PR body honestly names the cache invalidation chain
  (preserved-path stack → protocol dep → no cache → exposed CI failure)
  and the underlying error (`ERROR_DIRECTORY`, code 267 from
  `CreateProcessAsUserW`), and accurately notes that adjacent legacy
  tests already carry the same marker. That's the right framing for a
  CI-stabilization split-out from a larger feature branch.

## Risk

Negligible. Pure annotation, no runtime change. The test continues to
exist, can be run locally with `cargo test -- --ignored`, and the
`#[ignore = "..."]` reason flags it for follow-up. No production code
path is altered. The only thing this hides is a real bug that should
eventually get fixed — but the bug is already documented in the
ignore reason and is not a regression introduced by the merging branch.

The "ignore the test rather than fix the bug" pattern can become a
graveyard if not tracked. A follow-up issue or a comment with an
issue link in the ignore string would help, but this is a split-out
PR explicitly so reviewers can land the CI-stabilization independently
from the policy work that triggered the cache miss.

## Verdict

**merge-as-is** — exactly the right shape for a CI-stabilization split-out:
single-line, explicit reason, follows existing patterns in the file. The
right way to land "stop blocking the dependent stack on a flaky test
exposed by a refactor."

## What I learned

The cache-invalidation-exposes-cached-failure pattern is one of the
genuinely subtle CI failure modes — a refactor that doesn't touch the
failing test can still cause it to fail just by invalidating the build
cache that was hiding it. The honest framing in this PR ("the preserved
path stack invalidates the windows sandbox test target through its
protocol dependency, so this old test stops coming from cache and
exposes...") is exactly how to communicate that to reviewers without
making it sound like the policy stack introduced a bug.
