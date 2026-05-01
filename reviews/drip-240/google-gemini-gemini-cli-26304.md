# google-gemini/gemini-cli#26304 — Backlog Health & Stale Policy Optimization

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26304
- **Head SHA**: `3a06655ec5879deb17d506ccf8d62ebb37865fc9`
- **Verdict**: `needs-discussion`

## What it does

Three repository-process changes bundled as one PR by `gemini-cli[bot]`:

1. New `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts` (+64) that
   queries the GraphQL API for the oldest 100 open issues + PRs and
   emits median-age-in-days metrics.
2. `.github/workflows/stale.yml:43` raises `operations-per-run` from the
   default 30 to 200 (+1).
3. `.github/workflows/gemini-scheduled-stale-issue-closer.yml` (+58/-25)
   removes the `help wanted` permanent exemption and replaces it with a
   6-month grace window, plus refactors the closer to a two-phase
   "nudge-then-close" workflow with stale-label removal on revival.

## What's load-bearing

- **`gemini-scheduled-stale-issue-closer.yml:91-103`** — the meaningful
  policy change:
  ```
  // Skip if it has a maintainer or Public Roadmap label
  if (lowercaseLabels.some((l) => l.includes('maintainer')) ||
      rawLabels.includes('🗓️ Public Roadmap')) {
    continue;
  }

  // Special handling for 'help wanted'
  const isHelpWanted = lowercaseLabels.includes('help wanted');
  if (isHelpWanted && createdAt > sixMonthsAgo) {
    continue;
  }
  ```
  Removing the unconditional `help wanted` exemption means *any* `help
  wanted` issue older than 6 months becomes eligible for stale closure.
  This is a substantive community-policy change, not a pure cleanup.
- **`gemini-scheduled-stale-issue-closer.yml:135-186`** — the closer is
  flipped from "close on first detection" to two-phase: first run adds
  `batchLabel` + a "this will be closed in 14 days" comment; second run
  (when label is already present) closes with a different comment. Plus
  a third arm at `:175-186` that removes the stale label if activity
  resumed. This is a meaningfully kinder UX than the prior "close-on-
  detect" — but it's also a change in semantics that should be flagged
  in the PR body more visibly than "Throttling fix will increase the
  rate of stale item closure."
- **`stale.yml:43`** raising `operations-per-run` from 30 → 200 changes
  the rate-limit footprint of the daily cron by ~7×. Worth confirming
  this fits within the GitHub Actions API rate-limit budget for the
  repo size (the repo has ~2,300 open issues per the PR body, so 200/day
  means ~12 days for one full pass).
- **`backlog_age.ts:31-56`** — median-age calculation. The
  `(now - createdAt) / (1000 * 60 * 60 * 24)` produces days. The
  `ages.length % 2 !== 0 ? ages[mid] : (ages[mid - 1] + ages[mid]) / 2`
  median formula is correct. The `Math.round(... * 100) / 100`
  precision-trim outputs two decimals — fine.

## Concerns

1. **Author is `gemini-cli[bot]`** with no human reviewer signature on
   the commit. A PR that changes community-facing policy (deletes a
   permanent `help wanted` exemption) should be authored or at minimum
   co-signed by a human maintainer. The `## Backlog Health & Stale
   Policy Optimization` framing reads as bot-generated rationale, not
   as discussed maintainer policy. This needs explicit maintainer
   sign-off in the PR thread.
2. **No test coverage for `backlog_age.ts`.** The script has at least
   four failure modes (network error, empty result, mixed null nodes,
   GraphQL pagination cutoff) and zero unit tests. The
   `process.exit(1)` on error at `:62` will fail the workflow loudly,
   which is correct — but a test asserting that the median calculation
   gives `(ages[mid-1] + ages[mid]) / 2` for even-length arrays would
   be cheap and right.
3. **The two-phase nudge-then-close is good UX but the test for "stale
   label is removed when activity resumes" lives only in the YAML
   itself — no integration test.** This is the same shape as the
   community-policy concern: a 14-day grace window is meaningful and
   should be exercised at least once on a test repo before turning on
   in production.
4. **`backlog_age.ts:32` shells out via `execSync(\`gh api graphql -F
   owner=${GITHUB_OWNER} -F repo=${GITHUB_REPO} -f query='${query}'\`)`
   without escaping.** If `GITHUB_OWNER` or `GITHUB_REPO` ever come
   from untrusted sources (they're constants today, so safe in
   practice), the single-quoted query literal could break.
   Recommend `JSON.stringify` the env vars or use the GraphQL client
   library instead of shelling out.
5. **The 6-month threshold for `help wanted` expiration is asserted
   without a community discussion link.** "Help wanted is protected
   for 6 months" is a meaningful change to contributors' implicit
   social contract; please link to the issue/discussion where this
   threshold was decided (or open one for visibility).

## Verdict

`needs-discussion` — the technical changes are mostly fine, but a
bot-authored PR that deletes a permanent community-facing exemption
(`help wanted` no longer immortal) needs human maintainer
acknowledgment in the PR thread before merge. The `backlog_age.ts`
script is reasonable but should grow at least one test, and the
`operations-per-run: 200` bump should be confirmed against the repo's
Actions API budget. Once those are addressed, this is `merge-after-nits`.
