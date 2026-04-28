# openai/codex #20022 — ci: disable remote downloader without BuildBuddy

- PR: https://github.com/openai/codex/pull/20022
- Head SHA: `9811f5832c080a28e57b1f1ab1d1d386004a9582`
- Author: friel-openai
- Files: `.github/scripts/run-bazel-ci.sh` (+4/−2), `.github/scripts/test_run_bazel_ci.py` (+70/−0)

## Observations

- Root cause is precise and well-cited: Bazel's `--experimental_remote_downloader` *requires* gRPC remote caching to be configured (`The remote downloader can only be used in combination with gRPC caching`). When a fork PR runs without `BUILDBUDDY_API_KEY`, the existing fallback at `run-bazel-ci.sh:367-370` clears `--remote_cache=` and `--remote_executor=` but leaves the downloader pointed at the now-cleared gRPC endpoint, which produces the cited cross-cutting failure on PR #20020's Bazel and SDK jobs. The fix — adding `--experimental_remote_downloader=` to the same fallback array at `:380` — is the minimum diff that closes the inconsistency.
- The companion comment update at `:368-373` is good hygiene: it documents the *triple* of flags that together mean "force fully local" rather than leaving a future maintainer wondering why two of three remote-* flags are cleared. The Bazel docs URL is correctly linked for the new flag.
- The new pytest harness at `test_run_bazel_ci.py` is a nice example of black-box CI-script testing without standing up a real Bazel: `fake_bazel` shim writes its argv to a temp file, the test pops `BUILDBUDDY_API_KEY`, runs the real `run-bazel-ci.sh` with `PATH` redirected to the shim, and asserts the captured argv contains *all three* clears (`--remote_cache=`, `--remote_executor=`, `--experimental_remote_downloader=`) plus the startup option `--noexperimental_remote_repo_contents_cache`. This pins not just the addition but also the *position* of the new flag and the preservation of the existing args — a refactor that accidentally drops one of the other two clears would now fail.
- Argv assertion at `test_run_bazel_ci.py:84-95` is exact-list rather than subset: this means any new flag added to the fallback path will require updating the test, which is appropriate friction for a security-and-network-boundary script. The order is asserted, but Bazel doesn't care about order between these flags, so the strictness is over-pinned for behavioral correctness — fine for refactor-safety, slight nuisance for future contributors.
- `env.pop("BUILDBUDDY_API_KEY", None)` at `:65` correctly removes the key without erroring if it's already absent — important for local dev where the key is rarely set. The test runs end-to-end via `subprocess.run([...str(RUN_BAZEL_CI), "--", "build", "--", "//codex-rs/protocol:protocol"])` which exercises the real argument parsing in the shell script, so a regression in shell-quoting around the array expansion would also fail the test.

## Risks / nits

- The fake-bazel shim at `:51-58` uses `printf '%s\n' "$@" > {args_path}` which preserves whitespace but not arguments containing newlines. Not a real risk for Bazel flags, but worth a comment.
- No test for the **happy path** (when `BUILDBUDDY_API_KEY` *is* set) — only the local-fallback branch is exercised. A regression that accidentally added `--experimental_remote_downloader=` to the remote-enabled branch (clearing the flag the user actually wants) wouldn't be caught. Worth a parallel cell that sets `BUILDBUDDY_API_KEY=fake` and asserts the downloader flag is *not* in the argv.
- The PR's "Test plan" section says only `bash -n .github/scripts/run-bazel-ci.sh` (syntax check). The pytest harness is the actual validation but isn't mentioned. Worth updating the test plan in the PR body to credit the harness, since reviewers checking only the test plan would miss it.
- The script uses `--experimental_*` named flag — Bazel sometimes promotes these to unstable-named or removes the prefix. If/when Bazel renames `experimental_remote_downloader`, both the script and the test will need a coordinated update. A short comment naming the Bazel version this is correct against (e.g., "verified on Bazel 7.x") would help future maintainers.
- The PR title is fine but the body's "Failure addressed" section is more informative — readers scanning the title alone won't know this is a fix for a *cross-PR* failure (`PR #20020 Bazel job` and `sdk job`), not a fix for #20020 itself. The "unrelated failure observed on" framing is accurate but easy to misread.

## Verdict: `merge-as-is`

**Rationale:** One-line fix to a well-documented Bazel constraint, with an honest-to-goodness end-to-end CI-script test using a fake-bazel argv-capture shim. The test pins the full fallback argv shape, not just the new addition, so refactors can't silently drop one of the three clears. Nits are coverage-asymmetry observations (no positive-path test, no version anchor for the flag name) — none block merge for what is fundamentally a one-flag CI fix that closes a cross-PR failure pattern.

Date: 2026-04-29
