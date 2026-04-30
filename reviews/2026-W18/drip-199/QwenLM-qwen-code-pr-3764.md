# QwenLM/qwen-code PR #3764 — fix(ci): add merge-back PR for stable releases in release workflow

- PR: https://github.com/QwenLM/qwen-code/pull/3764
- Head SHA: `d2e60588f23345a05d936b60faf97e2f3ef4dc32`
- Files touched: 1 (`.github/workflows/release.yml` +34 / -0, net +34 lines).

## Specific citations

- Permission expansion at `.github/workflows/release.yml:307-308`: the existing release job's `permissions:` block (already had `contents: 'write'`, `packages: 'write'`, `id-token: 'write'`) gains `pull-requests: 'write'` and `issues: 'write'` to allow the new `gh pr create` and `gh pr merge --auto` calls to function.
- New step `Create PR to merge release branch into main` at `:414-433`: gated on the same three negations as the release tag itself — `${{ needs.prepare.outputs.is_dry_run == 'false' && needs.prepare.outputs.is_nightly == 'false' && needs.prepare.outputs.is_preview == 'false' }}`. So this step *only* fires for stable releases, never for nightly or preview or dry-run. Authenticates via `secrets.CI_BOT_PAT` (a separately-scoped bot PAT, not the workflow-default `GITHUB_TOKEN` — the latter wouldn't have write access to PRs in many org configurations).
- The PR-creation logic at `:421-432` is idempotent-ish: first does a `gh pr list --head "${RELEASE_BRANCH}" --base main --json url --jq '.[0].url'` lookup, and only creates a new PR if that lookup returned empty. PR title is `chore(release): ${RELEASE_TAG}`, body is `Automated release PR for ${RELEASE_TAG}. Syncs package.json versions on main.`. The resulting PR URL is captured into `$GITHUB_OUTPUT` as `PR_URL` for the next step.
- New step `Enable auto-merge for release PR` at `:435-444`: same gating predicate, runs `gh pr merge "${PR_URL}" --squash --auto --delete-branch` against the URL captured in the previous step. Uses `--auto` so the merge waits for branch-protection checks to pass before actually merging, and `--delete-branch` so the release branch self-cleans after the merge lands.

## Verdict: merge-after-nits

## Rationale

This is a clean, narrow CI workflow extension that closes a real ops gap: prior to this PR, the stable-release flow created a release branch, tagged it, published the GitHub Release, and then *left the release branch dangling* — meaning any version-bump commits that landed on the release branch never made it back to `main`, and the next release's release branch would be created from a now-stale `main` head. The PR adds the missing "merge release branch back to main" step using the standard `gh pr create` + `gh pr merge --auto --squash --delete-branch` recipe.

The gating is correct: the step only fires for stable releases (`is_dry_run == false && is_nightly == false && is_preview == false`), which matches the gate on the release-tag-creation step itself (presumably the same predicate above this in the same file, though that's not visible in the diff slice). Nightly/preview/dry-run releases don't need the merge-back because they don't bump on-disk version files in a way that needs to round-trip to `main`. This avoids the failure mode where a nightly run would try to PR a stale branch back and fail noisily every night.

The idempotency check (`gh pr list --head ... --base main --json url --jq '.[0].url'`) before the create is the right shape — if the workflow re-runs (or a previous attempt made it past `gh pr create` but failed on the auto-merge step), the second attempt will pick up the existing PR rather than creating a duplicate. `set -euo pipefail` at the top of both inline scripts means any failure surfaces as a workflow failure rather than getting swallowed.

The `--auto --squash --delete-branch` triplet is the canonical "fire-and-forget release-branch merge" pattern and is correct here: `--auto` waits for required status checks (so the workflow doesn't merge a broken release into `main`), `--squash` collapses the bump commits into a single audit-trail entry on `main`, and `--delete-branch` self-cleans the now-merged release branch so the branches list doesn't accumulate `release/v*` cruft.

Nits before merge:

1. **PAT-vs-default-token rationale not documented.** The step uses `secrets.CI_BOT_PAT` for `GITHUB_TOKEN` instead of the workflow-default `${{ github.token }}`. The reason is real (the default `GITHUB_TOKEN` won't trigger downstream workflow runs on the merge-back PR, which means required checks won't fire and the `--auto` will sit forever), but a one-line comment naming this rationale (`# CI_BOT_PAT used so the auto-merge PR triggers required checks; default GITHUB_TOKEN does not`) would prevent a future "let's standardize on github.token" cleanup from breaking the auto-merge.
2. **No fallback for the `gh pr list` lookup failure.** If `gh pr list` itself fails (transient gh-cli auth glitch), the `pr_url=$(...)` assignment captures empty and the script falls into the create-branch — which will then either succeed (fine) or fail with `--head already has a PR open` if the PR genuinely existed. Not a correctness bug, but the failure mode would be confusing in CI logs. A `pr_url="$(gh pr list ... 2>/dev/null || true)"` would suppress the false-positive error path.
3. **No commit-message body / breaking-change capture in the merge-back PR body.** The body is the literal string `"Automated release PR for ${RELEASE_TAG}. Syncs package.json versions on main."` — no list of what's actually being merged back, no link to the GitHub Release for the tag, no commit-range link. For a release-management audit trail, threading `${RELEASE_TAG}` into a `https://github.com/${owner}/${repo}/releases/tag/${RELEASE_TAG}` link in the body would close the loop.
4. **Permission elevation needs a sweep.** Adding `issues: 'write'` is for the `gh pr create` path (gh-cli's `create` actually doesn't need issues write — only the comment-on-PR path does), so this might be overscoped. A check that `pull-requests: 'write'` alone is sufficient for both `gh pr create` and `gh pr merge --auto` would let the diff drop the `issues: 'write'` line. Minor security-hygiene point — the GitHub Actions permission model rewards minimization.
5. **No regression test possible** — this is a CI workflow change and there's no test harness for `.github/workflows/*.yml` short of running it. A dry-run on a fork would be the validation.

None of these block merge — the change is correctly shaped, narrowly gated, and uses canonical patterns. They're polish.

## What I learned

The release-branch-back-to-main merge step is one of those "obvious in hindsight" gaps that a release pipeline can run without for years before someone notices the divergence between `main` and the latest stable tag. The right way to design the recipe — and this PR gets it right — is (a) gate on the stable-only predicate so dev/preview/nightly don't trigger noisy PR spam, (b) make the create idempotent so a re-run doesn't double-create, (c) use `--auto` so the merge respects branch protections rather than bypassing them, and (d) use a separately-scoped PAT rather than the default `GITHUB_TOKEN` so the resulting PR triggers required checks (the default `GITHUB_TOKEN`'s "no workflow trigger" property is a foot-gun specifically designed to prevent infinite recursion, but in this case it works against the goal). The one nit I'd push back on harder is the `issues: 'write'` permission — it's almost certainly overscoped, and CI permissions are exactly the kind of thing where "we added it because the docs example had it" tends to outlive the original need by years.
