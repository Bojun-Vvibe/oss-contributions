# BerriAI/litellm #26491 — Claude Code Compatibility Matrix v0

- **Repo**: BerriAI/litellm
- **PR**: [#26491](https://github.com/BerriAI/litellm/pull/26491)
- **Head SHA**: `28cdbb44858f1ec34a36824db9c65534bca1b9c9`
- **Author**: mateo-berri
- **State**: OPEN (+5604 / -1) — marked WIP
- **Verdict**: `needs-discussion`

## Context

Implements PRD #26476: a compatibility matrix between LiteLLM and
the Claude-Code-style CLI across providers (anthropic, bedrock,
vertex, azure). Two-track delivery:

- **PR gate** in CircleCI (`.circleci/config.yml:2006-2106`):
  installs the CLI at `latest minus 3 days` and runs the matrix
  on every PR.
- **Daily cron** in GitHub Actions
  (`.github/workflows/claude_code_compat_matrix.yml`): always-latest
  CLI, publishes JSON to a docs repo via a scoped GitHub App token.

## Design

Strong points:

1. **Trust separation is real, not just claimed**. The PR gate
   (CircleCI, trusted) uses `latest-minus-3-days` (config.yml:2026-
   2032 — a 3-day "security review buffer"). The cron (GH Actions,
   ephemeral runner) uses always-latest. A malicious CLI release
   only ever reaches the cron VM, never the trusted CircleCI fleet.
   This is exactly the right threat model and the comment at
   `claude_code_compat_matrix.yml:11-16` documents it explicitly.
2. **Cross-repo auth is minimal**. GitHub App scoped to
   `litellm-docs` only, `contents: write`, mints token at job-start
   (workflow lines 76-83). Token never enters the CLI's env.
3. **All third-party actions are pinned to commit SHAs**
   (`actions/checkout@08eba0b...`, `actions/setup-python@a26af69...`
   etc.) — supply-chain hygiene done right.
4. **Skip-publish escape hatch** (`workflow_dispatch` input
   `skip_publish`, lines 33-37) so operators can dry-run before
   committing.
5. **Release filter** (lines 47-50): only `v*-stable` tags trigger
   republish; `-rc1` and plain `vX.Y.Z` don't. Right policy.

## Risks / discussion-worthy

1. **5604 lines is a lot for a v0**. Slices 1-5 listed in the body
   were supposed to land independently; this PR appears to fold
   them all in. The PRD called out tracer-bullet → providers →
   gate → cron → full row set as separate PRs precisely so each
   could be reviewed in isolation. Reviewers will struggle.
2. **`set -euo pipefail` in the workflow** (line 80) is good, but
   the CircleCI step at config.yml:2042 (`CLAUDE_CODE_VERSION=$(uv
   run ... resolver)`) doesn't pipefail-guard — if the resolver
   exits 0 but prints empty, `npm install -g
   @anthropic-ai/claude-code@` becomes an open install of latest.
   Add `[ -n "$CLAUDE_CODE_VERSION" ] || exit 1` after line 2046.
3. **`compatibility-matrix.json` is gitignored** (`.gitignore:106`)
   in the source repo but pushed to `litellm-docs` by the cron.
   The `select_files_to_commit` enforcement that the workflow
   header mentions (line 25) needs to be visible in the
   `publisher` module — this PR includes the workflow but the
   reviewer can't easily see the enforcement code from the diff
   excerpt I have. Worth requesting a pointer in the PR description.
4. **30-minute `no_output_timeout`** on the pytest step
   (config.yml:2095). Realistic for 6×5 = 30 cells, but a single
   stuck provider will burn the whole budget. Consider per-test
   timeouts via `pytest-timeout`.
5. **`-e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY ... AWS_*`** passed
   directly to docker run (config.yml:2058-2069) — these end up in
   `docker inspect`. CircleCI env is fine for this, but a follow-
   up using `--env-file` from a tmpfs file would be tidier.

## Suggestions

- **Split this PR**. At minimum: (a) test fixtures + builder unit
  tests, (b) PR gate, (c) cron + publisher. Each is independently
  reviewable. The PRD ID structure (#26477-#26481) was already
  designed for this.
- Add the `[ -n "$CLAUDE_CODE_VERSION" ]` guard.
- Document the `select_files_to_commit` enforcement either in PR
  body or a CODEOWNERS-style comment in the publisher module.

## What I learned

The trust-boundary split between PR gate (3-day-aged CLI on a
trusted runner) and cron publisher (latest CLI on an ephemeral
runner) is a clean pattern for "we want to know if a third-party
release breaks us, but we don't want that release executing in
the same blast radius as our merge gate." Worth stealing for any
"daily check against external dep" workflow.
