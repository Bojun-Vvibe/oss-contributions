# QwenLM/qwen-code #3764 — fix(ci): add merge-back PR for stable releases in release workflow

- Link: https://github.com/QwenLM/qwen-code/pull/3764
- Head SHA: `d2e60588f23345a05d936b60faf97e2f3ef4dc32`
- State: MERGED 2026-04-30, +34/-0 across 1 file (`.github/workflows/release.yml`)

## Summary
After a stable release tag is cut, the version bump on the release branch
needs to land back on `main` so subsequent development sees the bumped
package versions. Previously this was a manual step (or not done at all,
producing skew). This PR adds two new workflow steps after the
`gh release create` step:

1. **Create PR to merge release branch into main** — uses `gh pr list`
   to detect an existing open PR from the release branch into main; if
   none, opens one with `gh pr create --base main --head <branch>` and
   a generated title `chore(release): <tag>`.
2. **Enable auto-merge for release PR** — `gh pr merge --squash --auto
   --delete-branch` so it merges itself once required checks pass.

Both steps are gated on `is_dry_run == 'false' && is_nightly == 'false'
&& is_preview == 'false'` so they only fire for real stable releases.

## Specific references
- `.github/workflows/release.yml:307-308` — adds new permissions
  `pull-requests: 'write'` and `issues: 'write'` to the job.
- `.github/workflows/release.yml:414-432` — "Create PR to merge release
  branch into main" step. Idempotent: re-runs detect the existing PR via
  `gh pr list --head "${RELEASE_BRANCH}" --base main`.
- `.github/workflows/release.yml:434-444` — "Enable auto-merge for release
  PR" step using `gh pr merge "${PR_URL}" --squash --auto --delete-branch`.

## Notes
- Idempotent PR creation is the right design — avoids the failure mode
  where re-running the workflow tries to open a duplicate PR.
- `set -euo pipefail` in both run blocks is good shell hygiene.
- Uses `secrets.CI_BOT_PAT` rather than the workflow's default
  `GITHUB_TOKEN`. This is necessary because PRs opened by the default
  token don't trigger workflows on `main` (so other CI checks would not
  run on the merge-back PR), but the PAT must be carefully scoped.
  Reviewer should confirm the PAT is restricted to this repo and to
  pull-requests scope only.
- The `chore(release): <tag>` title and the `--squash --auto
  --delete-branch` flags match common conventions; nothing surprising.
- Worth confirming: when `is_dry_run`/`is_nightly`/`is_preview` is true,
  these steps skip — but the prior `gh release create` step still runs
  with `${PRERELEASE_FLAG}`. Symmetry is correct.

## Verdict
`merge-after-nits` — confirm `CI_BOT_PAT` scoping and document the
expected behavior in the release runbook (so on-call knows the PR
auto-opens and auto-merges).
