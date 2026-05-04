# QwenLM/qwen-code PR #3685 — feat(sdk-python): add PyPI release workflow

- **PR:** https://github.com/QwenLM/qwen-code/pull/3685
- **Author:** doudouOUC
- **Head SHA:** `5c3af138` (full: `5c3af138ea1000d781fb379a97bad4863ce15cfa`)
- **State:** OPEN
- **Files touched:**
  - `.github/workflows/release-sdk-python.yml` (+515 / -0) — new release workflow
  - `packages/sdk-python/scripts/get-release-version.js` (+569 / -0)
  - `scripts/tests/get-release-version-python-sdk.test.js` (+930 / -0)
  - `.github/workflows/sdk-python.yml` (+2 / -0)
  - `packages/sdk-typescript/scripts/get-release-version.js` (+11 / -7)
  - `scripts/get-release-version.js` (+11 / -7)
  - `package.json` (+1 / -0), `package-lock.json` (+43 / -18)

## Verdict

**merge-after-nits**

## Specific refs

- `.github/workflows/release-sdk-python.yml:1-90` — workflow gates (`workflow_dispatch` only, `if: github.repository == 'QwenLM/qwen-code'`, `permissions: contents/id-token/issues/pull-requests: write`). The `id-token: write` is for PyPI Trusted Publishers (OIDC). Correct setup.
- `.github/workflows/release-sdk-python.yml:46-89` — the `Validate release inputs` step rejects: nightly+preview both true, malformed semver, ref/workflow_ref outside `main`, and `force_skip_tests` when `dry_run=false`. This is the right shape for a privileged release workflow — defense against a compromised contributor account triggering a release from a feature branch.
- `.github/workflows/release-sdk-python.yml:91-100` — every `actions/*` use is pinned to a 40-char SHA with a `# ratchet:` comment. Good. No floating tags.
- `scripts/tests/get-release-version-python-sdk.test.js` (+930) — substantial test surface for version computation (stable / preview / nightly / PyPI conflict / GitHub conflict). 930 LOC of test for 569 LOC of script is the right ratio for release tooling where bugs are catastrophic.
- `packages/sdk-python/scripts/get-release-version.js` (+569) — could not deeply audit in this pass; flag for closer review since this script gates the entire publish path.

## Rationale

The PR closes a real gap: docs reference `pip install qwen-code-sdk` but no PyPI release path exists. Workflow is well-engineered for a privileged release: SHA-pinned actions, branch-locked, environment-gated (`production-release`), explicit dry-run default. Test coverage on the version-computation script is generous.

Two nits worth raising before merge:
1. **Production guard.** The `dry_run` input defaults to `true`, which is correct, but consider also requiring an explicit `confirm: "yes-publish-{version}"` text input for non-dry-run runs to prevent fat-finger publishes. The `production-release` environment should already require manual approval, but belt-and-suspenders is cheap here.
2. **Test/script ratio caveat.** 930 LOC of test against a 569 LOC script suggests the script could be factored smaller; some of those tests may be exercising script-internal helpers that ought to be extracted into a stable, separately-versioned module. Not a blocker.

Solid, careful change. Worth merging once the production-confirmation prompt is considered.
