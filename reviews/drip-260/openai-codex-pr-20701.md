# openai/codex PR #20701 — ci: cross-compile Windows Bazel clippy

- PR: https://github.com/openai/codex/pull/20701
- Head SHA: `b6057de94b75103a4e061debc355808c24649938`
- Author: @bolinfest
- Size: +60 / -16
- Status: MERGED

## Summary

Switches the Windows Bazel clippy and verify-release-build CI jobs from `--windows-msvc-host-platform` (native Windows runner) to `--windows-cross-compile` (Linux RBE targeting `x86_64-pc-windows-gnullvm`). Both jobs are build-only signal — no test execution — so they don't need a native Windows host. Threads `--windows-cross-compile` through `run-bazel-query-ci.sh` and `list-bazel-clippy-targets.sh` so the target-discovery query selects the same `ci-windows-cross` config as the subsequent build. Fork PRs without `BUILDBUDDY_API_KEY` keep the old MSVC-host fallback inside `run-bazel-ci.sh`.

## Specific references

- `.github/workflows/bazel.yml:254-272` — clippy job: replaces `--windows-msvc-host-platform` with `--windows-cross-compile`, threads new `bazel_target_list_args` through `list-bazel-clippy-targets.sh`. The fork-fallback `--skip_incompatible_explicit_targets` is now gated on `[[ -z "${BUILDBUDDY_API_KEY:-}" ]]`, which is the right guard.
- `.github/workflows/bazel.yml:343-348` — verify-release-build: same swap, build-only justification documented inline.
- `.github/scripts/run-bazel-query-ci.sh:9-15,40-44` — new `--windows-cross-compile` flag selects `ci_config=ci-windows-cross`. Argument parsing uses an explicit while-shift loop, no third-party getopt — consistent with surrounding bash style.
- `scripts/list-bazel-clippy-targets.sh:8-21,33-43` — adds the same flag and forwards it into the query call. The if/else duplication of the bazel query block is mildly ugly but mechanically correct.

## Verdict: `merge-after-nits`

Right call: cross-compile is the documented fast path for these build-only jobs (#20585 already moved tests there), and the fork-fallback discipline keeps community PRs working. Verification commands in the PR description are concrete and reproducible (47 vs. 0 internal test binaries is exactly the assertion you want for "did target discovery use the right platform").

## Nits

1. **`list-bazel-clippy-targets.sh:33-50` duplicates the bazel-query call.** The if/else branches differ only by the `--windows-cross-compile` arg. Cleaner:
   ```bash
   query_args=(--output=label)
   [[ $windows_cross_compile -eq 1 ]] && query_args=(--windows-cross-compile "${query_args[@]}")
   manual_rust_test_targets="$(./.github/scripts/run-bazel-query-ci.sh "${query_args[@]}" -- 'kind(...)')"
   ```
   Avoids drift if the query expression changes.
2. **The fork-fallback gate `[[ -z "${BUILDBUDDY_API_KEY:-}" ]]` reads the secret directly, not whether `run-bazel-ci.sh` actually fell back.** If a fork has the secret set but BuildBuddy is unreachable, the fallback path runs without `--skip_incompatible_explicit_targets` and may hit the same incompatible-target failure that motivated the original flag. Probably fine in practice — BuildBuddy unreachability would fail the build for unrelated reasons first — but worth a comment.
3. **No CHANGELOG / job-rename note.** The clippy and verify-release-build job names are unchanged, but the underlying runner cost / characteristics shifted significantly (Windows runner minutes → Linux). If anyone tracks CI cost dashboards by job name, a heads-up in the PR description (already comprehensive) plus the PR title would help.
