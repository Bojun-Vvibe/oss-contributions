# block/goose #9025 — Switch GH pages deploy to actions/artifact workflow

- URL: https://github.com/block/goose/pull/9025
- Head SHA: `bc06fd0e959c9cf922a2697f1b07d98d8b1cb314` (`bc06fd0`)
- Author: @jamadeo (Jack Amadeo)
- Diff: +41 / −126 across 4 files
- Verdict: **merge-after-nits**

## What this changes

Migrates the documentation deployment pipeline from the
`peaceiris/actions-gh-pages` "commit to gh-pages branch" model over
to the official GitHub Pages artifact workflow
(`actions/configure-pages` + `actions/upload-pages-artifact` +
`actions/deploy-pages`). Three workflow files are rewritten and one
shell script (`scripts/clean-gh-pages.sh`, 39 lines) is deleted
outright. Net deletion of 85 lines.

Detailed touch list:

- `.github/workflows/deploy-docs-and-extensions.yml` (+17 / −32):
  drops `peaceiris/actions-gh-pages@v4`, the manual checkout of the
  `gh-pages` branch, the bespoke "preserve pr-preview directory"
  shell loop, and the `keep_files: false`/`force_orphan: true`
  scaffolding. Adds the canonical `permissions:` triplet
  (`contents: read`, `pages: write`, `id-token: write`) plus
  `environment: { name: github-pages, url: ${{ steps.deployment.outputs.page_url }} }`
  required by the new deploy action. Concurrency group renamed from
  `pr-preview` → `pages`.
- `.github/workflows/pr-website-preview.yml` (+7 / −31): the deploy
  side of the PR-preview workflow is **deleted entirely**. The job
  now only builds + verifies the docs map; the
  `rossjrw/pr-preview-action` deploy step is gone, and the entire
  separate `cleanup` job (which ran on `closed` PRs to scrub the
  preview from `gh-pages` via the deleted `clean-gh-pages.sh`) is
  also gone. Concurrency group narrows to per-PR
  (`docs-preview-${{ github.event.pull_request.number }}`) with
  `cancel-in-progress: true`.
- `.github/workflows/rebuild-skills-marketplace.yml` (+17 / −24):
  same shape of migration as the docs workflow — drops `peaceiris`,
  adds the configure/upload/deploy chain, swaps to the `pages`
  concurrency group and the `github-pages` environment.
- `scripts/clean-gh-pages.sh` (−39): obsolete because the
  `gh-pages` branch is no longer the deploy target.

## Why this is the right migration

The artifact-workflow path is the model GitHub itself recommends and
it has three concrete operational advantages over the
commit-to-gh-pages approach: (1) deploys are atomic — the user never
sees a half-deployed gh-pages branch — because the artifact upload
and the deploy are distinct steps and the deploy is transactional;
(2) deploy history is captured natively in the GitHub Pages
"Deployments" UI with rollback affordances, instead of being
shadowed in the gh-pages branch's git log where rollback means
force-pushing a previous SHA; (3) the `id-token: write` permission
plus `environment: github-pages` enables OIDC-based deploy
authentication which is materially more secure than the `secrets.GITHUB_TOKEN`
push-to-branch model.

The concurrency-group cleanup is correct too: the previous
`pr-preview` group co-mingled production deploys, PR previews, and
the marketplace rebuild on the same serialization point with
`cancel-in-progress: false`, which meant a slow PR preview could
delay a hotfix docs push for tens of minutes. The new layout puts
production deploys + marketplace on a shared `pages` group (correct
— they both publish to the same Pages site) and PR previews on a
per-PR group with `cancel-in-progress: true` (correct — when a PR
gets a force-push, the in-flight preview build is wasted work and
should be cancelled).

## Concerns

(a) **PR previews are silently disabled.** The biggest behavioral
change — and the one most easily missed reading the diff — is that
the `pr-website-preview.yml` workflow no longer deploys anywhere. It
builds, it verifies, but the deploy step is gone. That's
*intentional* given the migration (the artifact workflow doesn't
support the `gh-pages/pr-preview/<n>` subdirectory model that
`rossjrw/pr-preview-action` provided), but it means contributors who
relied on the auto-comment "preview at https://.../pr-preview/123/"
on their PRs lose that affordance entirely. The PR body's `## Summary`
is empty and `## Related Issues` is the unfilled template — the
deliberate removal of PR previews deserves a sentence in the body
and ideally a follow-up issue tracking "re-add PR previews via X
mechanism (e.g. Cloudflare Pages, Netlify, or a separate ephemeral
staging environment)."

(b) **No `concurrency.cancel-in-progress` rationale on the production
group.** The new `pages` group keeps `cancel-in-progress: false`,
which is correct (you don't want a second push to the docs cancel
the first push's deploy mid-upload), but the prior file had a 4-line
comment explaining the choice and the new file doesn't. Re-add the
comment so the next maintainer doesn't "fix" it.

(c) **`actions/configure-pages` requires Pages to be configured for
the repo with source = `GitHub Actions` (not "Deploy from a
branch").** This is a one-time settings flip in the repo
configuration that the workflow can't do for itself. Without it the
deploy step fails with `HttpError: Not Found`. Worth either
(a) calling out in the PR description that the maintainer needs to
flip Settings → Pages → Source before merging, or (b) running the
workflow once on a branch and confirming green before landing on
main.

(d) **Action SHA pins are fresh and not yet in any other workflow
in the repo.** `actions/configure-pages@983d7736...`,
`actions/upload-pages-artifact@7b1f4a76...`, and
`actions/deploy-pages@d6db9016...` are all new dependencies for this
repo. Consider adding them to the project's Dependabot config (or
equivalent) so they get rotated on the same cadence as the existing
`actions/setup-node` and `actions/checkout` pins.

(e) **The deleted `scripts/clean-gh-pages.sh` and the deleted
`cleanup` job were the only path to ever shrink the `gh-pages`
branch.** That branch will continue to exist with its accumulated
historical preview deploys in it. Worth a follow-up housekeeping PR
that either deletes the `gh-pages` branch entirely (now that nothing
deploys to it) or at least force-pushes it to a single empty commit
so the repo's `.git` size stops carrying the legacy preview history.

The migration is the right move and the workflow file is well-shaped.
The five nits above are mostly maintainer-checklist items rather than
code-review fixes — once the Pages source is flipped in repo
settings and the PR-preview removal is acknowledged in the body, this
is good to ship.
