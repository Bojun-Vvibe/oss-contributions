# Review — google-gemini/gemini-cli#26355

- PR: https://github.com/google-gemini/gemini-cli/pull/26355
- Title: Robust Scale-Safe Lifecycle Consolidation
- Head SHA: `6bb4a497aa996284efc8f7706161486dc60dbad2`
- Size: +169 / −626 across 7 files (all under `.github/workflows/`)
- Verdict: **merge-after-nits**

## Summary

Consolidates four separate "stale issue", "stale PR", "no-response",
and "scheduled-triage" GitHub Actions workflows into a single new
`gemini-lifecycle-manager.yml` cron, and removes the older split
files. Net −457 LOC of CI plumbing. Uses a daily `30 1 * * *` cron,
supports `workflow_dispatch` with a `dry_run` boolean input, and
ratchets the `actions/create-github-app-token` and
`actions/github-script` action versions to pinned commit SHAs.

## Evidence

- `.github/workflows/gemini-lifecycle-manager.yml:1-166` (new file) —
  single workflow with `permissions: { issues: write, pull-requests:
  write }`, `concurrency: { group: '${{ github.workflow }}',
  cancel-in-progress: true }`, repo-guard
  `if: github.repository == 'google-gemini/gemini-cli'`.
- Same file, ~lines 40-90 — inline `actions/github-script` body
  defines `STALE_LABEL = 'stale'`, `NEED_INFO_LABEL =
  'status/need-information'`, `EXEMPT_LABELS = ['pinned',
  'security', '🔒 maintainer only', 'help wanted', '🗓️ Public
  Roadmap']`, with thresholds `STALE_DAYS=60`, `CLOSE_DAYS=14`,
  `NO_RESPONSE_DAYS=14` — these match the prior split files.
- Removed (whole-file deletes, contributing the −626 line count):
  - `.github/workflows/gemini-scheduled-issue-triage.yml`
  - `.github/workflows/gemini-scheduled-stale-issue-closer.yml`
  - `.github/workflows/gemini-scheduled-stale-pr-closer.yml`
  - `.github/workflows/no-response.yml`
  - `.github/workflows/pr-contribution-guidelines-notifier.yml`
  - `.github/workflows/stale.yml`
- Pinned action SHAs (good practice):
  - `actions/create-github-app-token@fee1f7d63c2ff003460e3d139729b119787bc349`
  - `actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea`

## Notes / nits

- `concurrency.group: '${{ github.workflow }}'` with
  `cancel-in-progress: true` plus a daily cron is fine, but the
  `workflow_dispatch` path will *cancel* an in-flight scheduled run.
  If someone clicks "Run workflow" mid-run, the cron's progress is
  lost. Either drop `cancel-in-progress` for this workflow or scope
  the group to `${{ github.workflow }}-${{ github.event_name }}`.
- `processItems` calls `github.rest.search.issuesAndPullRequests`
  with `per_page: 100` and no pagination loop ("batch limited" is
  literally what the log says). For a repo with >100 stale items
  on day one of activation, a non-trivial tail will be skipped each
  run. A `for await (const page of github.paginate.iterator(...))`
  loop or an explicit "iterate while items.length === 100" would
  remove the silent ceiling.
- `EXEMPT_LABELS` includes `'🗓️ Public Roadmap'` and `'🔒 maintainer
  only'` — confirm these emoji-prefixed names match the actual
  label slugs in the repo (label name match in the GitHub search
  API is case-sensitive on the slug). A quick `gh label list` cross-
  check before merge would catch any drift.
- The `dry_run` boolean is wired through `env.DRY_RUN` and read as
  `process.env.DRY_RUN === 'true'`; ensure every mutating
  `github.rest.issues.*` call inside the script honors it (not
  inspected line-by-line here — worth a grep before merge).

Direction is correct; tighten the pagination and the concurrency
scope before landing.
