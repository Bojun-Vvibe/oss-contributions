# openai/codex#20585 — ci: cross-compile Windows Bazel tests

- **PR**: https://github.com/openai/codex/pull/20585
- **Head SHA**: `f9dd4e40751f21493b1dfb437480843766ae3657`
- **Size**: +269 / -66, 12 files
- **Verdict**: **merge-after-nits**

## Context

Native Windows Bazel PR CI baseline on `main` ran 28m12s against a 30-minute timeout (run `25206257208/job/73907325959` per PR body), leaving ~2 min headroom. PR mirrors the macOS Bazel CI shape (build remotely on Linux RBE, run tests locally on the platform runner) for Windows. This is intentionally separate from #20485 (Cargo cross-compile PoC) — this PR only changes Bazel PR CI.

## What's right

**`.bazelrc:153-167` — new `ci-windows-cross` config block.**

Twelve-line block that:
- Inherits from `ci-windows` (`--config=ci-windows`) so all existing Windows-side defaults apply.
- Layers `--config=remote` and `--host_platform=//:rbe` so build actions hit RBE.
- Uses `--strategy=remote` for general actions but `--strategy=TestRunner=local` so test execution stays on the Windows runner — exactly the pattern macOS already uses (per `common:ci-macos --strategy=TestRunner=darwin-sandbox,local` at `:155`).
- Sets `--platforms=//:windows_x86_64_gnullvm` (the actual Rust target triple for Windows binaries) and `--extra_execution_platforms=//:rbe,//:windows_x86_64_msvc` (RBE for build actions, MSVC-host for local test execution).
- Adds `--extra_toolchains=//:windows_gnullvm_tests_on_msvc_host_toolchain` — the test-toolchain bridge so gnullvm-built test binaries can execute under the MSVC host platform that Bazel's test wrapper (`tw.exe`) selection requires.

**`.github/scripts/run-bazel-ci.sh:265-296` — host-platform / shell-executable / PATH plumbing.**

Three closely-related blocks of post-rc-expansion overrides:

1. **`:281-289` — `--host_platform=//:rbe --shell_executable=/bin/bash` re-applied after rc expansion when `BUILDBUDDY_API_KEY` is set.** The PR-body justification is exactly right and worth quoting verbatim: `--enable_platform_specific_config` expands `common:windows` on Windows hosts *after* ordinary rc configs, so `ci-windows-cross`'s RBE host platform gets clobbered unless re-asserted on the command line. Without `--shell_executable=/bin/bash`, Bazel derives the genrule shell from the Windows client and remote Linux actions get asked to run `C:\Program Files\Git\usr\bin\bash.exe` — the exact failure mode CI run `25208908640/job/73915086362` exhibited (failed at 4m49s with V8 genrule path errors).
2. **`:291-296` — fork-PR fallback to `--jobs=8` when `BUILDBUDDY_API_KEY` is missing.** Fork PRs don't receive the BuildBuddy secret, so the cross-compile path can't work; this gates back to local execution with the existing low-concurrency cap rather than failing CI for fork contributors. Correct fail-safe.
3. **`:330-358` — Windows `*_action_env` plumbing now branches on `pass_windows_build_env`.** When cross-compile is on with BUILDBUDDY available, the `INCLUDE`/`LIB`/`UCRTVersion`/etc. MSVC env-var dump is *skipped* for build actions (those run on Linux RBE where MSVC env is meaningless), but `--test_env=PATH=${CODEX_BAZEL_WINDOWS_PATH}` is preserved unconditionally so locally-executing tests still have the Windows PATH. The `--action_env=PATH=/usr/bin:/bin` and `--host_action_env=PATH=/usr/bin:/bin` for the cross-build branch correctly normalize the remote-build PATH.

**`.github/scripts/compute-bazel-windows-path.ps1:49-50,87-92` — MinGW runtime DLL paths added to the cache-stable PATH.**

Two new arms accept `C:\mingw64\bin` and `C:\msys64\*\bin` from the existing PATH walk, plus an explicit `Add-StablePathEntry` loop covering `mingw64\bin`, `msys64\mingw64\bin`, `msys64\ucrt64\bin`. Correct because gnullvm-built test binaries link against MinGW runtime DLLs that aren't in the MSVC PATH.

**`run-bazel-ci.sh:114-117` — failed-target extraction now also matches `ERROR: .* Testing //... failed:`** in addition to the prior `^FAIL: //` shape. Cross-compile changes the error path Bazel takes for some failure modes, and the prior regex was missing failures emitted via the `Testing` channel.

## Risks / nits

- **Wrapper selection is implicit on the test-toolchain bridge.** PR body explains it correctly: Bazel selects the Windows test wrapper (`tw.exe`) from the test action's *execution platform*, not from the Rust target triple, so `windows_gnullvm_tests_on_msvc_host_toolchain` is what actually makes test execution land on the MSVC-host runner. A one-line comment in `.bazelrc` next to the `--extra_toolchains=` line explaining this load-bearing-but-non-obvious behavior would prevent a future CI cleanup from "simplifying away" the bridge toolchain.
- **PR has not yet posted CI evidence on the split branch.** Body says "the fresh PR CI run on this split PR is the one that should provide the final end-to-end timing." Worth gating merge on at least one green PR-CI run on this exact branch with timing data, since the prior runs were on the pre-split branch and one of them (`73915086362`) already revealed the V8 genrule shell-path issue. Without a fresh green run, the 30-minute timeout headroom claim is unverified for this PR.
- **`--local_test_jobs=8`** at `.bazelrc:161` mirrors the fork-PR fallback's `--jobs=8`, but the comment at the fallback site only justifies the latter. A one-line note that `local_test_jobs=8` is the *test* concurrency cap (separate from build job count) under cross-compile would help a future operator tuning these caps.
- **`pass_windows_build_env=1` default at `:332`** is the conservative choice (preserves prior behavior when not on the cross-compile path), but the surrounding control flow has now grown to four branches (cross+BUILDBUDDY, cross-no-BUILDBUDDY, non-cross, non-Windows). Worth a small comment block at `:330` or a helper function `compute_windows_action_env_args(...)` to keep future maintainers from breaking the matrix accidentally.
- **V8 patches** (mentioned in PR body but not visible in head of diff) are the highest-risk surface — generated tools, generated sources, Windows assembly assumptions. The PR keeps standalone `//third_party/v8:all` out of ordinary Bazel CI as before, but transitive consumers under `//codex-rs/...` still build through V8. Worth confirming in PR body which specific V8 patches were touched and whether the existing V8-only CI workflow still passes against them.
- **`.github/workflows/bazel.yml:25,54`** — minor cleanup: removes the dead "we need true distributed builds" comment and renames the job from "Local Bazel build" to "Bazel test on ${{ matrix.os }}". Both correct cosmetic-but-load-bearing changes since the job now genuinely is no longer "local" on Windows.

## Verdict

**merge-after-nits.** Mirror-the-macOS-pattern is the right shape, the `--host_platform` / `--shell_executable` re-application after rc expansion is the load-bearing detail that earlier attempts (the `73915086362` run) got wrong, and the fork-PR fallback correctly preserves the existing local path for contributors without BuildBuddy access. Need a fresh green PR-CI run on the split branch with timing evidence and a one-line comment on the test-toolchain bridge before merge.
