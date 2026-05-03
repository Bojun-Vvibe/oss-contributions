# BerriAI/litellm PR #27062 — restore ghcr_deploy.yml workflow

- Link: https://github.com/BerriAI/litellm/pull/27062
- SHA: `7a06b724e56a6531ece8cb84eff2d6b3cca015ff`
- Author: Pratiknarola
- Stats: +460 / 0, primary file `.github/workflows/ghcr_deploy.yml` (new, 433 lines)

## Summary

Adds (or re-adds) the `Build, Publish LiteLLM Docker Image. New Release` workflow at `.github/workflows/ghcr_deploy.yml`. The workflow is `workflow_dispatch`-only with `tag`, `release_type`, and `commit_hash` inputs, and runs three Docker-Hub push jobs (main image, `litellm-database`, `litellm-spend_logs`, `litellm-non_root`) plus a GHCR multi-arch build-and-push job. PR body is a near-empty template (no Linear ticket, no CI run links, no test plan, no screenshot).

## Specific references

- `.github/workflows/ghcr_deploy.yml` L1–L13: `workflow_dispatch` with required `tag` and `commit_hash`, default `release_type: latest`. Fine, but `release_type` is consumed only by the `print` job — it never gates which `tags:` get pushed. So choosing `dev` or `rc` produces the same images as `latest`. Either wire it into the tag list or drop the input.
- L40–L45: `if: github.repository == 'BerriAI/litellm'` on `docker-hub-deploy` is correct — prevents forks from publishing on accident. The downstream `build-and-push-image` job is missing this guard, so a fork dispatch would still try to push to `ghcr.io/<fork>/litellm` using the fork's `GITHUB_TOKEN`. Add the same guard.
- L55–L70: `tags: litellm/litellm:${{ github.event.inputs.tag || 'latest' }}` — the `|| 'latest'` is dead code because `tag` is `required: true`. Harmless, but if `release_type` is meant to drive the tag (e.g. push both `${tag}` and `stable`), this is where it would happen.
- L75–L100 (database/spend-logs/non_root sub-images): no `platforms:` arg, so these end up linux/amd64 only, while the GHCR `build-and-push-image` job sets up QEMU + buildx for multi-arch. Inconsistent matrix is a release-quality issue — ARM users who pull `litellm-database:<tag>` won't get an arm64 image.
- L104–L115: `docker/login-action` and `docker/metadata-action` are pinned by full commit SHA (`65b78e6e…`, `9ec57ed1…`) — good supply-chain hygiene. But the Docker-Hub job (L60–L70) uses floating `@v3` / `@v5` tags. Pin both jobs the same way.
- No `concurrency:` block. Two near-simultaneous dispatches with the same `tag` will race each other on the registry. Add `concurrency: { group: ghcr-${{ inputs.tag }}, cancel-in-progress: false }`.
- PR body has zero context: no link to whatever issue/incident caused this workflow to be re-added, no diff against the previous version that was removed, no CI run link. For a 433-line workflow that touches release plumbing, that's not enough to land.

## Verdict

verdict: request-changes

## Reasoning

The workflow itself is mostly conventional, but four real issues have to be fixed before it's safe to merge: (1) `release_type` input is never consumed beyond an `echo`, so the contract is misleading; (2) the GHCR job lacks the `if: github.repository == 'BerriAI/litellm'` fork guard that the Docker-Hub job has; (3) the helper images don't get multi-arch builds even though the main image does; (4) action pins are inconsistent across jobs. Plus the PR body has no rationale for restoring this file. None of these are hard, but releasing through a workflow with these gaps is the kind of thing that bites at exactly the wrong moment.
