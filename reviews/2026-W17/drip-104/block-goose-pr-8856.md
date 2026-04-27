---
pr: https://github.com/block/goose/pull/8856
sha: 6a8bb14c8f08747974e115fb0215c1f07857053e
diff: +40/-0
state: OPEN
---

## Summary

Adds a new GitHub Actions workflow `.github/workflows/dependabot-auto-merge.yml` that, on every dependabot PR open/synchronize, fetches the dependabot metadata and — if the update type is `version-update:semver-patch` or `version-update:semver-minor` — auto-approves the PR and enables auto-merge with the `--merge` strategy. Major-version bumps are intentionally left alone.

## Specific observations

- `.github/workflows/dependabot-auto-merge.yml:14-16` — gates on `github.event.pull_request.user.login == 'dependabot[bot]' && github.repository == '<owner>/goose'`. The repository-name guard is the right defense against this workflow accidentally activating on a fork that gets pushed back to the canonical remote, but it pins the canonical owner-name as a string literal — any future repo rename or org migration silently disables auto-merge with no failure signal. A repo variable would be more robust but is overkill for a 40-line workflow.
- `:18-22` — uses `dependabot/fetch-metadata@d7267f607e9d3fb96fc2fbe83e0af444713e90b7` pinned by full SHA, not by tag. Good supply-chain hygiene; matches the project's apparent dependabot-friendly stance toward its own pin policy.
- `:24-30` and `:32-38` — the approve and enable-auto-merge steps share the same gate predicate (`patch || minor`) duplicated verbatim. Cheap to factor into a `needs:` step that exports a single `should_auto_merge` output, but the duplication is honest and keeps each step independently grep-able.
- `:9-11` — `permissions:` block grants `contents: write` and `pull-requests: write`. Both are required (`gh pr review --approve` needs PR-write, `gh pr merge --auto --merge` needs PR-write + repo-write to actually merge). No `id-token` granted — correct, this workflow doesn't need OIDC.
- Strategy is hardcoded to `--merge` (creates a merge commit). For a project that may prefer squash-merge for PR history hygiene, this should be a configurable variable or at minimum documented. Most dependabot rollups (per drip-101 #8855 and drip-103's various dep-bumps) on this repo go in as merge commits in the visible history, so `--merge` matches existing convention.
- No automatic-rebase step before merge — if a dependabot patch PR sits behind a few main commits and CI requires up-to-date branches, auto-merge will block on "branch is out of date" with no remediation. The dependabot-rebase command is on by default for the bot, so this is usually self-healing within an hour, but worth a comment.
- Does **not** require any specific status check to pass before merging — relies on whatever branch protection rules are configured on `main`. This is the right design (workflow shouldn't duplicate branch-protection logic) but means a misconfigured branch protection would silently auto-merge unverified dependabot PRs. Worth one-time verification by a maintainer that branch protection on `main` requires CI green before this lands.

## Verdict

`merge-after-nits` — well-scoped 40-line workflow with correct permission scoping, appropriate dependabot-bot author check, full-SHA pin on the action dependency, and the right semver-major exclusion. Two cheap follow-ups: (1) one-line comment naming branch-protection-on-main as the load-bearing safety net since this workflow doesn't enforce its own checks, (2) confirm `--merge` vs `--squash` matches project preference (current dep PRs go in as merges, so likely fine).
