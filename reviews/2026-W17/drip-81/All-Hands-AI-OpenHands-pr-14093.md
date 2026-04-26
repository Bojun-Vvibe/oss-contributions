# All-Hands-AI/OpenHands PR #14093 — chore(deps): bump actions/setup-python from 5 to 6

- **PR:** https://github.com/All-Hands-AI/OpenHands/pull/14093
- **Author:** app/dependabot
- **Head SHA:** `a815ad2c108b`
- **Stats:** +6 / -6 across 4 files (`.github/workflows/{lint-fix,lint,py-tests,pypi-release}.yml`)
- **Verdict:** `merge-as-is`

## What this changes

Mechanical Dependabot bump of the `actions/setup-python` GitHub Action from major version 5 to major version 6 across all four CI workflows that consume it: `lint-fix.yml:75`, `lint.yml:50` and `:67`, `py-tests.yml:48` and `:83`, and `pypi-release.yml:27`. The diff is six identical `@v5` → `@v6` changes with no other workflow edits.

`actions/setup-python@v6` (the GA release in early 2026) is a small-but-real upgrade over v5 with three changes worth knowing about: it bumps the runtime from Node 20 to Node 24 (matching GitHub's broader Node-24 migration on hosted runners), it switches the default `update-environment` behavior to also expose `pythonLocation` consistently across self-hosted and hosted runners, and it drops fallback support for the EOL Python 3.7 line (which the OpenHands matrix doesn't use anyway — `py-tests.yml` runs Python 3.12 and `pypi-release.yml` is also pinned at 3.12). None of these surface in OpenHands' usage.

## Why merge-as-is

All six call sites pass either `python-version: 3.12` (4 of them) or `python-version: ${{ matrix.python-version }}` (2 of them in `py-tests.yml`), and all four workflows already use `cache: "pip"` or `cache: "poetry"` — both of which v6 supports unchanged. There's no `update-environment: false` override anywhere, no reliance on the deprecated `version-file` shape, and no use of the removed `architecture` defaults. The cache-restore key format is unchanged across v5 and v6, so cached pip/poetry directories from prior runs will continue to hit on the first post-merge build.

The Node-24 runtime jump is the only thing that could realistically misbehave, and only for a workflow doing something exotic with Action-internal-state — which a vanilla `setup-python@v6` step doesn't expose. Hosted Linux runners (`ubuntu-latest`) are already on `actions/runner` images that bundle Node 24, so there's no agent-side prerequisite either.

The drive-by note that `pypi-release.yml:26` already uses `actions/checkout@v6` (one major version ahead of the typical `@v4`) tells us this repo is comfortable on the latest action majors, so the bump matches established policy. Dependabot's choice not to group this with other action bumps is mildly annoying — `actions/checkout@v5`, `actions/cache@v5`, `pypa/gh-action-pypi-publish@v1` updates would all sit naturally with this — but that's a Dependabot config question for the OpenHands repo, not a reason to block this PR.

## What I'd want to verify if I were running it

Just kick CI on the PR and watch the four affected workflows go green: `lint`, `lint-fix`, `py-tests` (matrix), and the no-op rendering of `pypi-release` (which only fires on tag pushes — the PR-level run will skip the publish step but should still pass the syntax/setup phase). If all four are green, ship it.

## Bottom line

Six-line major-version action bump with zero functional reach into OpenHands' code. Standard low-risk dependabot merge.
