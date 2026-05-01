# Review: google-gemini/gemini-cli #26302 — Backlog Health & Stale Policy Optimization

- URL: https://github.com/google-gemini/gemini-cli/pull/26302
- Head SHA: `5328faff2fcc41424e68e817e8a745d0e4e3c11e`
- Files: 3 (`.github/workflows/stale.yml`, two new metric scripts under
  `tools/gemini-cli-bot/metrics/scripts/`)
- Size: +162/-21

## Summary of intended change

Three repository-process changes bundled in one PR, motivated by 2342 open
issues + 440 open PRs:

1. **Stale workflow split** at `.github/workflows/stale.yml`: replaces the
   single `stale` job (one matrix runner, both issues + PRs) with two
   separate jobs:
   - `stale-issues` (`stale.yml:10-37`): `permissions: issues: write`,
     `only-issues: true`, `operations-per-run: 300`,
     `exempt-issue-labels` extended with
     `needs-triage,waiting-on-maintainer,status: blocked`.
   - `stale-prs` (`stale.yml:39-50`): `permissions: pull-requests: write`,
     `only-prs: true`, `operations-per-run: 200`, `exempt-pr-labels`
     extended with the same three new labels.
2. **`backlog_health.ts`** (new, +84): paginates the GraphQL API to fetch
   up to 500 of the *oldest* open PRs and issues (5 pages × 100 ordered
   `CREATED_AT ASC`), computes median age in days, emits two metric lines
   `backlog_median_age_pr_days,X.YY` and `backlog_median_age_issue_days,X.YY`.
3. **`stale_ratio.ts`** (new, +51): single GraphQL query for total open
   counts plus stale-labeled counts via `search`, emits ratios.

## Review

### Stale workflow split — necessary for the permissions tightening

The split is the right shape. Pre-PR, the single `stale` job needed both
`issues: write` and `pull-requests: write` for the GITHUB_TOKEN, so a
compromised `actions/stale` invocation could touch either surface. Post-PR,
each job has the minimum permission needed for its surface. That's a real
security tightening even apart from the policy changes.

The `concurrency` block from the pre-PR file is dropped at `:23-25`. That
was previously
`group: '${{ github.workflow }}-stale', cancel-in-progress: true` and
prevented overlapping runs (e.g. cron + workflow_dispatch firing
simultaneously). With the split into two jobs, neither has a concurrency
block, so a concurrent dispatch could race with a scheduled run on the
same surface. **Recommend** restoring concurrency per-job:
```yaml
concurrency:
  group: ${{ github.workflow }}-stale-issues  # or -stale-prs
  cancel-in-progress: true
```

The matrix-runner setup from pre-PR is dropped, which is fine since both
jobs hardcode `runs-on: ubuntu-latest`. The matrix was previously a
single-element list anyway.

### Operations-per-run jump

Default is 30. Pre-PR was implicitly default. Post-PR: issues 300, PRs 200.
That's a ~17× increase on issues and ~7× on PRs. Each "operation" against
`actions/stale` consumes ~2-4 GraphQL points, so the worst-case Actions API
budget for one cron tick goes from ~120 points to ~2000 points. Probably
fine against the 5000/hr GraphQL limit, but worth confirming in the PR
body, especially if the cron schedule fires more than hourly.

### Exempt-label list extension

Adds `needs-triage,waiting-on-maintainer,status: blocked` to both
exempt-issue and exempt-pr labels. All three are sensible — issues
explicitly waiting on the team should not be auto-staled. Note that
`status: blocked` (with the colon-space separator) needs to literally
match the label name in the repo; if the repo uses `status:blocked` (no
space) this won't match and will silently no-op. Quick check of the
repo's label list before merge.

### `backlog_health.ts` script

The math is **correct**:
- Median formula at `:48-56` handles both even and odd lengths
  (`mid = Math.floor(n/2); even ? (ages[mid-1] + ages[mid])/2 : ages[mid]`).
- Pagination at `:32-43` correctly walks up to 5 pages, breaks early on
  `!hasNextPage`, accumulates `totalCount` from the latest page (which is
  always the same — `totalCount` is the unfiltered open count).
- Stderr warning at `:62-67` honestly reports when the median is computed
  on a sampled subset (oldest 500 of N).

Issues:
- **Shell injection risk** at `:34-37`: `execSync` with template literal
  interpolating `GITHUB_OWNER`, `GITHUB_REPO`, and `cursor`. The first two
  are imports from `../types.js` and presumably hardcoded, so likely safe
  *today*, but `cursor` is the GraphQL `endCursor` returned by GitHub —
  attacker-controlled if a malicious PR somehow embeds shell metacharacters
  in the cursor. Cursors are opaque base64 in practice, but principled fix
  is to switch to `gh api graphql --field cursor=$CURSOR` style or
  shell-out via `JSON.stringify`-quoted env vars. Same shape applies to
  `stale_ratio.ts:24` where `GITHUB_OWNER` and `GITHUB_REPO` are
  interpolated directly into the GraphQL `search` query string — if either
  ever held a backtick or quote, parse would break.
- **Median of "oldest 500"** is a *bounded* worst-case signal, not a true
  backlog median. If 2342 open issues and the oldest 500 are all >2yr old,
  median = ~3yr; this is correctly described as "worst-case signal" in
  the PR body, but the metric *name*
  `backlog_median_age_issue_days` doesn't carry that caveat. Consider
  renaming to `backlog_oldest500_median_age_issue_days` or document
  prominently in the script header.
- **No tests**. For a metric script that will drive policy decisions, at
  least one unit test of the median formula on a known input array
  (especially even-length arrays) would catch off-by-one regressions.
  Pure function, easily testable.

### `stale_ratio.ts` script

- The GraphQL query at `:11-25` mixes a parameterized `repository(owner,
  name)` block with two `search(...)` queries that interpolate
  `${GITHUB_OWNER}/${GITHUB_REPO}` directly into the search string —
  inconsistent style. Ideally both forms parameterize through `-F`.
- The `label:stale OR label:Stale` syntax at `:18` and `:21` is using
  GitHub search OR — confirm GitHub treats this as logical-OR not
  literal-string-search. Better explicit: a single `label:stale` plus
  enable case-insensitive label matching in repo settings.
- No empty-repo guard: `totalIssues > 0 ? ... : 0` correctly handles
  divide-by-zero. Good.

## Verdict

**needs-discussion**

The high-level direction (split jobs, tighten permissions, expand
exemptions, add observability) is right. Specific items needing
maintainer attention before merge:

1. **Restore per-job concurrency blocks** — the split lost the
   pre-PR `cancel-in-progress` guard, opening a race between
   scheduled and dispatched runs on the same surface.
2. **Confirm the `operations-per-run: 300/200` jump** doesn't break
   the GraphQL API budget if the cron fires more than hourly.
3. **Verify the `status: blocked` (with space) label literally exists
   in the repo's label set** — typo in the exempt list silently no-ops.
4. **Add at least one unit test** for the `getMedianAgeDays` function
   in `backlog_health.ts` (even-length input is the bug-prone case).
5. **Tighten shell-injection surface** in both scripts: switch
   GraphQL-query string interpolation to `gh api graphql --field`
   parameter passing, or at minimum quote inputs via `JSON.stringify`.
6. **Clarify metric-name caveat**: `backlog_median_age_*_days` is
   actually "median of oldest 500", which is a different statistic
   from the true backlog median. Either rename the metric or document
   the sampling bias in the script header and downstream dashboards.
7. **Bot-authored PR**: this is a substantive policy change to a
   community-facing automation. Needs an explicit human maintainer
   sign-off in the PR thread before merge — not just CI-green.
