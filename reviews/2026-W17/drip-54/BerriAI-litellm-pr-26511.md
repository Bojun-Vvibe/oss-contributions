# BerriAI/litellm PR #26511 — ci: add supply-chain guard to block fork PRs that modify dependencies

@0bd9213d · base `litellm_internal_staging` · +140/-0 · author `krrish-berri-2`

## Summary
New GitHub Actions workflow that blocks fork PRs from modifying `uv.lock` or adding new packages to any `pyproject.toml`. Designed to harden against dependency-confusion / supply-chain attacks via untrusted PRs.

## What changed
- `.github/workflows/guard-fork-dependencies.yml` (new, 140 lines) — single job `guard` gated by `if: github.event.pull_request.head.repo.full_name != github.repository`.
- Triggers on `pull_request` against `main`, `litellm_internal_staging`, `litellm_oss_branch`, and `litellm_**` glob, scoped to `paths: [uv.lock, pyproject.toml, litellm-proxy-extras/pyproject.toml, enterprise/pyproject.toml]`.
- Embeds an inline Python script (heredoc, lines 58-102) that PEP 508-parses `[project].dependencies`, `optional-dependencies`, and `dependency-groups`, then diffs the normalized name set base→PR.

## Key observations
- `permissions: {}` at the job level (line 22) — excellent, correctly applies least-privilege.
- Pinned action SHAs (`actions/checkout@08eba0b27e820071cde6df949e0beb9ba4906955`) — also correct hardening posture.
- `persist-credentials: false` on both checkouts (lines 35, 41) — prevents the embedded `GITHUB_TOKEN` from being available to subsequent steps. Good.
- The `uv.lock` check is a strict bytewise diff (`diff -q`, line 47); fine, but the failure message could include `git diff base/uv.lock pr/uv.lock --stat | head` to make triage easier.
- The Python script is run via `python3 /tmp/extract_deps.py` — relies on `python3` being on the runner image (true for `ubuntu-latest`, but worth a `setup-python` step for stability).
- Edge case: dependency *removals* are silently allowed (only `comm -13`, line 124 = "lines only in PR"). Removal-by-fork seems benign but worth noting in the workflow comment.
- Doesn't catch `[tool.uv.sources]` or git/url overrides that don't change the `pyproject.toml` dependency *names* — a fork could repoint an existing dep to a hostile git URL. Consider extending `extract_dep_names` to also diff `[tool.uv.sources]` and `[tool.poetry.dependencies]` URL specs.

## Risks/nits
- No test coverage of the workflow itself; checklist in the PR body claims tests added but the diff shows none. Maintainers should confirm.

**Verdict: merge-after-nits**
