# block/goose #8956 — chore(deps): bump actions/github-script from 8.0.0 to 9.0.0

- **PR**: https://github.com/block/goose/pull/8956
- **Head SHA**: `4325f4ec6b353a09605af58d7e7d573edda01e4e`
- **Author**: dependabot[bot]
- **Size**: +8 / −8, 6 files

## Files changed

All `.github/workflows/*.yml`, all bumping the same pinned SHA for
`actions/github-script`:

- `code-review.yml` (1 occurrence)
- `pr-comment-build-cli.yml` (1)
- `pr-comment-bundle.yml` (3)
- `recipe-security-scanner.yml` (2)
- `update-hacktoberfest-leaderboard.yml` (1)
- `update-health-dashboard.yml` (1, inferred from PR file list)

Old pin: `actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # v8`
New pin: `actions/github-script@3a2844b7e9c422d3c10d287c895573f7108da1b3 # v9.0.0`

## Specific-line review

- The pinning strategy (`<owner>/<action>@<full-sha> # vX`) is correct
  and is preserved across every file. No comment is dropped, no SHA
  format is changed. Dependabot's behavior here is exactly right.
- v8 → v9 of `actions/github-script` is a major bump; per the
  upstream release notes the headline change is the bundled Node
  runtime moving to Node 22 (was Node 20) and dropping support for
  Node 20-only syntax. Every consuming workflow in this PR is using
  `script: |` blocks with modern JS — none rely on Node 20-specific
  behaviors I can spot from the diff context.
- Of the six workflows touched, two are security-sensitive:
  - `pr-comment-bundle.yml` and `pr-comment-build-cli.yml` use
    `github-script` for the "verify commenter has repo write access"
    gate that prevents fork PRs from triggering CLI builds.
  - `recipe-security-scanner.yml` uses it to post scan results and
    set commit status checks.
  A maintainer should confirm the `script:` bodies still execute
  cleanly under the v9 runtime — no API-shape surprises from `octokit`
  or `core` modules. The bodies aren't shown in this diff (only the
  `uses:` lines change), so the change itself is mechanically safe;
  the real risk lives in the unchanged surrounding code.

## Risk

Low-to-moderate. Mechanical pin bump, but on a security-critical
gating action. CI on this PR (if it runs the affected workflows)
will be the real signal.

## Verdict

**merge-after-nits** — let CI/required checks run on this PR
specifically (especially `pr-comment-bundle.yml` and
`recipe-security-scanner.yml` jobs) and confirm the v9 runtime
executes the existing `script:` blocks before merging. If checks
are green, this is safe to merge as-is.
