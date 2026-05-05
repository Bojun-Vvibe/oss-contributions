# openai/codex #21259 — ci: trigger rusty-v8 releases from tags

- URL: https://github.com/openai/codex/pull/21259
- Head SHA: `cef1ce37b9ee112c244d7bf32f6d92809aa3dd2a`
- Author: cconger
- Size: +10 / -27 in `.github/workflows/rusty-v8-release.yml`

## What it does

Switches the `rusty-v8-release` workflow trigger from `workflow_dispatch` (with optional `release_tag` and `publish` inputs) to `push` on tags matching `rusty-v8-v*.*.*`. The release tag is now derived from `github.ref_name` rather than from a manual input, and the workflow asserts at `release_tag` resolution time that the pushed tag matches the in-tree v8 crate version (`expected_release_tag = "rusty-v8-v${V8_VERSION}"`), `exit 1`-ing on mismatch. Removes the now-redundant `if: ${{ inputs.publish }}` gate on the `publish-release` job and the manual "ensure publishing from default branch" guard step.

## Observations

- **Tag-version drift guard is the load-bearing change.** The previous flow let an operator manually dispatch with any `release_tag`, with no tie-back to the actual v8 crate version on the checked-out ref. The new `release_tag != expected_release_tag → exit 1` at `:43-47` ensures the GitHub release name and the published artifact's reported v8 version cannot diverge. Good.
- **Lost guardrail: branch check.** The deleted "Ensure publishing from default branch" step at `:148-156` used to refuse to publish from non-default branches. Tag pushes can come from *any* branch (a tag is a ref independent of branches), so dropping this guard means a `rusty-v8-v1.2.3` tag pushed from a feature branch would now publish. The implicit replacement is "the version-match check at `:43-47` is enough" — which is true *if* nobody can fast-forward a feature branch to a v8-version-bumping commit and tag it before the default-branch PR lands. Worth confirming with maintainers whether tag-protection rules cover this, or whether an explicit `if: github.event.repository.default_branch in (refs of github.sha)` check is wanted.
- **Concurrency group widens slightly.** Previously `${{ inputs.release_tag || github.run_id }}` (per-run unless explicit tag), now `${{ github.ref_name }}` (per-tag). This is the right model — two runs for the same tag should serialize — and `cancel-in-progress: false` is preserved.
- **Implicit operator UX loss.** No more "click Run workflow → optionally toggle publish=false for a dry-run" path. If maintainers want a dry-run mode for testing the build pipeline against a tag without publishing, they'll need to either re-add `workflow_dispatch` as a parallel trigger or split `build` and `publish-release` into two workflows. The PR description doesn't mention whether dry-run was actually used in practice.
- The `concurrency.group` reference at `:8` correctly picks up the tag name; tags pushed from a delete-and-recreate sequence would still serialize properly.

## Verdict: merge-after-nits

Net simplification with a real correctness improvement (tag↔version match assertion). Two concerns worth a maintainer ack before merge: (1) loss of the default-branch publish guard relies on tag-protection rules being in place, and (2) loss of a manual dry-run path may bite future debugging. Neither blocks merge if maintainers have already reasoned through these, but they should be called out explicitly in the PR description or a follow-up README note.