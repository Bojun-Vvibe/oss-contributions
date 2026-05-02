# BerriAI/litellm #26997 — chore(deps): bump the github-actions group across 1 directory with 18 updates

- **Head SHA:** `0f9a523d5190fcf89f5567c35bd91a45a4dba7af`
- **Files:** 34 workflow files under `.github/workflows/` — all SHA-pinned action bumps
- **Verdict:** `merge-as-is`

## Rationale

Dependabot group bump touching 34 workflow files. Every change is a SHA-pin update with the comment-pin convention preserved (e.g. `actions/checkout@de0fac…` # v6.0.2, `actions/setup-python@a309ff…` # v6.2.0, `actions/cache@27d5ce…` # v5.0.5, `actions/upload-artifact@043fb4…` # v7.0.1, `codecov/codecov-action@57e3a1…` # v6.0.0). Pinned SHAs are the security-correct way to consume third-party actions and the comment annotations match the resolved tags.

Worth noting: `actions/upload-artifact` v4 → v7 is a multi-major bump and v5 changed default artifact retention/compression behavior — the existing usage in `_test-unit-base.yml` (`name: coverage-...`, `path: coverage.xml`) is plain enough that the v7 defaults shouldn't affect it, but CI runs on this PR are the actual proof. Same for `codecov/codecov-action@v6` which changed token-handling semantics; the current call uses `use_oidc: true` which is the v6-blessed path, so this is fine.

If CI is green on the PR, merge as-is. No human-meaningful behavior changes; this is exactly the kind of bump that should land quickly to keep the supply-chain pin window short.

