# openai/codex#21057 — bazel: run sharded rust integration tests

- **URL**: https://github.com/openai/codex/pull/21057
- **Head SHA**: `99b12d60f6c0`
- **Diffstat**: +122 / -49
- **Verdict**: `merge-after-nits`

## Summary

Restores Bazel test coverage for sharded Rust integration tests on macOS. The repo runs Bazel with `--noenable_runfiles`, but `rules_rust`'s native sharding wrapper requires a symlink runfiles tree; in that configuration the wrapper would fail to find the test binary, emit an empty test list, and exit `0` — silently green. Switches sharded integration targets to manual `*-bin` binaries dispatched through the existing manifest-aware `workspace_root_test` launcher, while keeping Bazel's stable test-name sharding on the launcher itself.

## Findings

- `defs.bzl` (+88 / -29) — core change. Sharded targets such as `//codex-rs/core:core-all-test` now wrap a `*-bin` rust_binary inside `workspace_root_test`, which is already manifest-aware and works without runfiles symlinks. Verified by author with `bazel query --output=build //codex-rs/core:core-all-test` showing the wrapping. Reviewer should confirm the launcher still propagates `--test_filter` / `--test_env` correctly to the inner binary (the example invocation in the PR body shows `--test_filter=suite::request_permissions_tool::approved_folder_write_request_permissions_unblocks_later_apply_patch` working, which is good evidence).
- `.bazelrc` (+7 / -3) — adds `CODEX_BAZEL_TEST_SKIP_FILTERS` plumbing. Reasonable, but the env-var-driven skip list is a soft contract; a comment in `.bazelrc` linking to the launcher source that consumes it would help future readers.
- `.github/workflows/bazel.yml` (+17 / -17) — sets `CODEX_BAZEL_TEST_SKIP_FILTERS=suite::code_mode::` to keep V8/code-mode tests out of Bazel CI, and excludes the standalone `codex-rs/code-mode` and `codex-rs/v8-poc` unit-test targets from the workflow. PR body is explicit that Cargo CI remains responsible for V8 coverage — that boundary should be documented in `codex-rs/code-mode/README.md` (or wherever Cargo-vs-Bazel coverage is tracked) so the gap doesn't get re-forgotten.
- `scripts/list-bazel-clippy-targets.sh` (+10 / -0) — small helper, low risk.
- The PR cross-references a real bug uncovered by re-enabling coverage: `approved_folder_write_request_permissions_unblocks_later_apply_patch`. The fix is correctly split into a separate PR (#21060), which keeps this PR scoped to the build-system change. Good hygiene.

## Nits

- `defs.bzl` — confirm the new `*-bin` targets are tagged `manual` so they don't accidentally get pulled into `bazel test //...` wildcards (PR body says "manual `*-bin` binaries" but the diff snippet doesn't show the tag — worth a reviewer eyeball).
- The previous silent-pass behavior is exactly the kind of CI failure that should never happen again. A regression test that asserts a deliberately-failing test in `core-all-test-bin` actually fails the Bazel target would lock this in. Not blocking, but valuable.

## Recommendation

Restores real coverage and cleanly splits out the bug it surfaced. Address the manual-tag confirmation and add a brief comment on the V8/Cargo-only boundary, then merge.
