# google-gemini/gemini-cli #26093 — chore/release: bump version to 0.41.0-nightly.20260428.gc17400b83

- PR: https://github.com/google-gemini/gemini-cli/pull/26093
- Author: gemini-cli-robot (release automation)
- Head SHA: `e3542e397d20`
- Diff: 19+/19- across 9 files (root `package.json` + `package-lock.json` + 7 workspace `package.json` files)

## Verdict: merge-as-is

## Rationale

- **Pure mechanical version bump** from `0.41.0-nightly.20260423.gaa05b4583` to `0.41.0-nightly.20260428.gc17400b83`. The new version string encodes a 5-day later nightly date (`20260428` vs `20260423`) and a different commit SHA suffix (`gc17400b83` vs `gaa05b4583`). Diff is 9× the same one-line `version` field bump plus the matching `package-lock.json` updates (4 lock entries for the root package + 7 for each workspace, all consistent).
- **Lockstep across all 9 workspaces:** `packages/a2a-server`, `packages/cli`, `packages/core`, `packages/devtools`, `packages/sdk`, `packages/test-utils`, `packages/vscode-ide-companion`, plus the root and the lockfile. No drift — every workspace pins to the exact same nightly version, which is the invariant nightly bumps must preserve. A drifting bump (e.g. `core` at one version, `sdk` at another) is the failure mode this PR demonstrates the absence of.
- **`sandboxImageUri` updated symmetrically at `package.json:17` and `packages/cli/package.json:30`** to the matching `sandbox:0.41.0-nightly.20260428.gc17400b83` registry tag. The two `sandboxImageUri` references in the repo are the only spots where the version appears outside `version` fields, and both got bumped. This is the load-bearing co-bump — a release where the sandbox image tag drifts from the npm version produces a working CLI that runs against a stale sandbox image, which is the worst kind of release-time bug because it doesn't fail at install or first-run.
- **Bot author (`gemini-cli-robot`) and PR body "Automated version bump for nightly release"** — this is the standard, recurring release-cut shape. Has correct provenance (committed by the release automation account), correct scope (only version-string fields), correct shape (lockfile + all workspaces in one atomic diff). No human review value beyond verifying these three properties, which the diff confirms.

## Nits / follow-ups

- The `package-lock.json` diff shows 9 `version` updates (root + 7 workspaces + lockfile self-reference at `:18077`) which matches what `npm install` would produce locally. No transitive-dep changes leaked into this PR — a clean dep-graph-snapshot bump. If the lockfile bump had pulled in unrelated transitive updates, that would be the foothold for a supply-chain regression; it didn't.
- Out of scope for this PR but worth noting the broader pattern: nightly version-bump PRs are a high-trust target for typosquat / supply-chain attacks because reviewers tend to skim the lockfile diff. The 19+/19- symmetric file-count is a good pre-screen heuristic — anything other than that exact ratio (one `version` line up + one down per file) on a "chore/release: bump version" PR deserves a closer look.
- No CHANGELOG entry — but this is `nightly`, not a release-line cut, so a CHANGELOG bump per nightly would be noise. The next non-nightly release will roll up commits between this nightly and the previous one.

## What I learned

Mechanical-bump release PRs are the most boring possible reviews and that's exactly what makes them dangerous: the only bug that can hide here is something *not* shaped like a version bump, and the way to catch that is by counting (file count, line count, lockfile entries) before reading the diff. A 9-file PR with 19+/19- where every line is `"version": "..."` going up by exactly the same delta is the strongest possible signature of a clean automated bump. Anything else — extra files, asymmetric lockfile changes, transitive-dep updates that weren't in the previous nightly's diff — is a foothold and deserves human investigation regardless of who the author claims to be. The structural lesson is that "automation produced exactly the diff it always produces" is the load-bearing invariant for trusting bot-authored release PRs at all.
