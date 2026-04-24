---
pr_number: 19389
repo: openai/codex
head_sha: 92a97081f023fb31b7759a9ef9efe166458a35c3
verdict: merge-after-nits
date: 2026-04-24
---

# openai/codex#19389 — Guard npm update prompt on registry readiness

**What changed.** Three coordinated changes in `codex-rs/tui/src/updates.rs`, `update_action.rs`, and `update_prompt.rs`:

1. New `VersionSource` enum (`GithubRelease`/`HomebrewCask`/`NpmRegistry`, line 343) tags the cached `version.json` with where the version came from. Old caches without a `source` field are trusted for github/brew but not for npm (`matches_source`, line 335).
2. `UpdateAction` gains `NpmGlobalVersion(String)` and `BunGlobalVersion(String)` (lines 73–79); `with_target_version` (line 87) pins npm/bun installs to a verified version. The variants drop `Copy` because they now own a `String`, which is why the diff sprinkles `.clone()` calls in `cli/src/main.rs`, `tui/src/app.rs`, and `update_prompt.rs`.
3. Most importantly, npm's "ready" check (`npm_ready_latest_version`, line 478) verifies that `dist-tags.latest` actually has the platform-specific `optionalDependencies` entry pointing at the matching `@<scope>/codex-<os>-<arch>@<version>-<tag>` and that **that** version's tarball is published. Stops the popup from telling users to install a version where their platform's binary hasn't propagated yet.

**Why it matters.** npm registry ingestion is not atomic — root manifest can be live before per-platform packages publish. Without this gate, the TUI tells users to run an install that will fail or pull a stale binary.

**Concerns.**
1. **`current_npm_platform_package` panics on unsupported tuples via `anyhow::bail!`** (line 473). That bubbles up as `Err` from `check_for_update` and the spawn task logs it — fine, but it means an unrecognized arch will show *no* update prompt forever. Worth a fallback to `GithubRelease` for those arches.
2. **`HashMap<String, NpmPackageVersionInfo>`** (line 367) deserializes the *entire* npm registry response, which for a popular package is ~megabytes and grows monotonically. `serde_json` will allocate every version. Consider streaming or fetching the dist-tag-pinned manifest URL instead (`/{pkg}/{version}`) to keep this O(1) in published versions.
3. **Source mismatch wipes `dismissed_version`?** No — `prev_info.and_then(|p| p.dismissed_version)` (line 188) preserves it. Good. But if a user switches install method (npm → brew) and dismisses, then switches back, the dismissal is honored across sources — likely surprising. Acceptable for now, worth a TODO.
4. **`#![cfg(any(not(debug_assertions), test))]`** on `updates.rs` (line 269) plus `#[cfg_attr(test, allow(dead_code))]` is the cleanest way to test prod-only modules. Nice.
5. Tests (`npm_ready_latest_version_*`, lines 605–649) cover the three failure modes; missing one for the `NpmGlobalVersion(_)` -> `with_target_version` idempotence path. Cheap to add.

Targeted fix to a real "false positive update prompt" class. Land after a thought on the registry-payload size and unsupported-arch fallback.
