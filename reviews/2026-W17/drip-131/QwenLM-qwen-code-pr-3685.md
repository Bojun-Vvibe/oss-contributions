# Review — QwenLM/qwen-code #3685

- **Title**: feat(sdk-python): add PyPI release workflow
- **Author**: doudouOUC (jinye)
- **Head**: `56a350233aff729b46656bb66c6b5b7e842cb0af`
- **Verdict**: merge-after-nits

## Summary

Adds a 330-line GitHub Actions workflow `.github/workflows/release-sdk-python.yml` plus `packages/sdk-python/scripts/get-release-version.js` to build, validate, and publish `qwen-code-sdk` to PyPI via trusted publishing. Supports stable/preview/nightly release types with PyPI/GitHub conflict checks. Closes the gap between documented `pip install qwen-code-sdk` and actually having a publishable release pipeline.

## Findings

- **Trusted publishing via `pypa/gh-action-pypi-publish@release/v1`** is the right modern pattern — no API token in repo secrets, OIDC-based auth via `id-token: 'write'` permission. PR body correctly names the required PyPI-side configuration (Owner/Repository/Workflow/Environment tuple) and acknowledges this must be set up out-of-band before the workflow can publish.
- **Concurrency group `${{ github.workflow }}` + `cancel-in-progress: false`** at `:34-36` correctly serializes release runs without canceling an in-flight publish — a canceled `pypa/gh-action-pypi-publish` mid-upload could leave PyPI in a partial state. Right call.
- **Action ratchet pinning** — `actions/checkout@08c6903cd...` (v5), `actions/setup-node@49933ea5...` (v4), `actions/setup-python@a26af69b...` (v5) all pinned to commit SHAs with version tags as comments. This is the security-best-practice shape; `pypa/gh-action-pypi-publish@release/v1` at `:198` is the only un-SHA-pinned action — pin it for consistency, or document why publish is the exception.
- **`if: github.repository == 'QwenLM/qwen-code'` guard at `:39-40`** correctly prevents fork CI from attempting to publish — would fail at the OIDC step but better to short-circuit.
- **`force_skip_tests` input default `false`** at `:30-32` is the right default; the PR body explicitly notes "Production releases should keep this disabled". Consider gating `force_skip_tests=true` behind an additional `dry_run=true` requirement (`if: ${{ inputs.force_skip_tests != 'true' || inputs.dry_run == 'true' }}`) so a real publish can never skip the smoke test.
- **`get-release-version.js` validated for all three release types** per PR body — nightly returns `0.1.1.dev20260428023154` (PEP 440 dev release), preview returns `0.1.1rc0` (PEP 440 release candidate), stable returns `0.1.0`. All three are valid PEP 440 versions PyPI will accept; the JS-script approach to deriving Python versions is unusual but justified by needing to share git/GitHub-tag conflict-check logic with the Node side.
- **`pypa/gh-action-pypi-publish` with `skip-existing: true`** at `:201-202` is the right idempotency posture — re-runs of the same release tag won't fail on "version already exists" — but combined with `dry_run` semantics there's a subtle gap: a dry-run that sets `dry_run=false` accidentally and publishes once, then is re-run with `dry_run=true`, won't re-publish (good) but also won't warn that the prior accidental publish happened. Consider an explicit `gh release view` check before publish that would surface the prior-publish state.
- **`OPENAI_API_KEY`/`OPENAI_BASE_URL`/`OPENAI_MODEL` in the smoke-test env at `:158-160`** uses real credentials for a release-time smoke test — appropriate for catching regressions but means every release-workflow run consumes API quota and a production-grade key must be in `production-release` environment secrets. Document the quota expectation.
- **`packages/sdk-python/dist` is regenerated via `rm -rf dist`** at `:181-184` immediately before `python -m build` — clean shape, but the `Show publish artifacts for dry-run` step at `:208-211` only `ls -la`s the result. Add a `pip install` smoke against the local wheel to catch packaging-metadata bugs before the real publish.
- **Stable-release branch creation at `:214-227`** uses `git switch -c "release/sdk-python/${RELEASE_TAG}"` and pushes the version-bump commit to that branch. The `--follow-tags` push at `:243` will push the release tag if `git tag` was run earlier — but I don't see explicit `git tag` in the visible diff window. Verify the tag is created somewhere downstream (likely in a follow-on step beyond the 250-line slice fetched).

## Recommendation

Merge after (a) pinning `pypa/gh-action-pypi-publish` to a SHA for ratchet consistency, (b) gating `force_skip_tests=true` behind `dry_run=true`, (c) adding a local-wheel `pip install` smoke step before publish, and (d) confirming the release-tag creation path is wired (visible diff shows the push but not the tag creation). The architectural shape is sound — trusted publishing, ratcheted actions, conservative concurrency, and the dual stable/preview/nightly support is the right granularity.