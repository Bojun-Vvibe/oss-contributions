# block/goose #8875 — chore(release): release version 1.33.0

- PR: https://github.com/block/goose/pull/8875
- Author: github-actions (bot)
- Head SHA: `18751404194c`
- Diff: 0+/0- (release-cut PR — no file changes; serves as the merge anchor for tag push)

## Verdict: merge-as-is

## Rationale

- **Standard release-PR shape, not a code-changing PR.** The PR body contains release instructions (`git tag v1.33.0 origin/release/1.33.0 && git push origin v1.33.0`) that the maintainer runs to actually cut the release; the PR itself auto-closes once the tag is pushed. No diff to review — the review surface is the *cherry-pick set* listed in the PR body that constitutes v1.33.0.
- **Cherry-pick set covers ~45 commits** spanning a coherent feature/fix/dep-bump mix:
  - Feature: cross-worktree kill (#8768), `--check` flag for `goose info` (#8289), session metadata storage migration to backend (#8769), ACP server smaller refactor (#8787), Windows CUDA release artifacts (#8750), NVIDIA provider + declarative provider UX (#8798), removed-extensions handling (#8797), goose2 signed release flow (#8728), skills library redesign (#8868)
  - Fix: starting new session in project after creating one (#8766), `_meta` instead of `meta` in newSession (#8796), draft-PR stale-check (#8803), only-cleanup-on-same-repo (#8799), removed deprecated provider tests (#8801), missing underscore in `updateWorkingDir` (#8743), CI flaky smoke-test timeout (#8837), login-shell PATH probe suspending goose on startup (#8804), preprompt excluded from session title generation (#8793), defaults YAML removed (#8854), `OPENAI_BASE_URL` fallback alias (#8069), lazy local inference runtime in router (#8656)
  - Dep bumps: cargo group of 23 deps (#8855), individual `winreg` / `rcgen` / `lopdf` / `anstream` / `tiktoken-rs` / `openssl` bumps, four GitHub Actions bumps, two npm doc-deps bumps
  - Release plumbing: bump-to-1.33.0 (#8872), open-release-branch commit (`187514041`), TUI/SDK 0.19.0 sub-release (#8806), Windows CUDA download links hidden until release (#8874)
- **All commits trace to PRs that merged to `main` first** (per the PR body's "All commits in this release should have corresponding cherry-picks in `main`" reminder), which is the load-bearing invariant for release-branch hygiene — every release commit is a cherry-pick of a main commit, never a release-branch-only commit, so `main` stays the source of truth. This PR's existence on `release/1.33.0` and the listed commit SHAs are evidence the cherry-picks landed; the maintainer's job is to spot-check that no release-branch-only fixups crept in.
- **The drip-140 reviewer has previously flagged commits in this set:** `#8870` (cumulative token usage in stream-json/json complete event) was reviewed in drip-136 with a `merge-after-nits` verdict; #8857 (preserve thinking content for providers that require it) was reviewed earlier. These have already cleared individual review and so don't need re-review at the release-cut step.

## Nits / follow-ups

- **Mixed-content release ("feat" + "fix" + "chore" + 11 dep bumps) tagged as `1.33.0` (minor)** — semver-correct since at least one `feat` commit is present, but the changelog rendering will be busy. The PR body lists commits in merge order rather than grouped by Conventional-Commit type; downstream changelog tools (release-please etc.) handle this, but humans reading the PR body have to mentally re-bucket. Worth a follow-up to either auto-group in the PR body or rely entirely on release-tooling-rendered changelogs.
- **`#8855 cargo-minor-and-patch group with 23 updates`** is the largest single dep-bump line and the most likely supply-chain footgun. The release-cut PR can't see what's in the group — that's a separate dependabot PR. If the maintainer hasn't already verified the group on `main`, they should at least check that `cargo audit` / `cargo deny` were green on the post-#8855 commit before tagging.
- **`#8728 goose2 signed release flow`** introduces signing — worth confirming the signing key and CI plumbing for that flow is in place on the `release/1.33.0` branch, not just `main`. Cherry-picks of signing setup commits are easy to get wrong because they're cross-cutting (CI config + secret refs + cargo.toml metadata).
- **`#8854 fix: remove defaults yaml`** is the kind of small-titled fix that can have user-visible defaults regression — worth a release-note callout if any user-facing defaults shifted, since the title alone doesn't say what was lost.

## What I learned

Release-cut PRs are the inverse of code-changing PRs: the diff is empty, but the *commit-set listed in the body* is the actual review object. The right shape is to sample-check three classes of commits — (1) any feat commit that introduces user-visible API changes, (2) any large dep-bump group, (3) any release-plumbing commit that touches CI/signing — and trust that all `fix` commits already passed individual review on main. The "all release commits are cherry-picks of main commits" invariant is what lets a release-cut PR be approved without re-reviewing the full diff: it reduces the trust question to "did the cherry-picks apply cleanly and did anything get added that isn't on main?", which is `git log release/1.33.0..main` style verification. The pattern generalizes: any time you have a release branch and a development branch, the release-PR review should verify *symmetry* (release ⊆ main) rather than *content* (which was already verified upstream).
