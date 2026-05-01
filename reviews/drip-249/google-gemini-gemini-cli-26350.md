# google-gemini/gemini-cli #26350 — Fix Metrics History Retention & Accuracy

- URL: https://github.com/google-gemini/gemini-cli/pull/26350
- Head SHA: `6ec068c2ab3cc324ed1c9cf67330d5c1ed13ddd5`
- Files: `tools/gemini-cli-bot/metrics/index.ts` (+4/-4), 2 new scripts (`backlog_age.ts` +48, `bottlenecks.ts` +74), `tools/gemini-cli-bot/metrics/scripts/throughput.ts` (+56/-16)

## Context / problem

The repository's `gemini-cli-bot` metrics pipeline writes a CSV time series to a rolling-window file. Four observed issues being fixed in one PR:

1. **Window too small**: 100 rows held only ~1.5 runs of data at the current cron cadence — not enough to compute 7d/30d deltas, defeating the purpose of a time series.
2. **No backlog-age signal** — visibility into "are we accumulating old issues" was missing.
3. **`throughput.ts` conflated `authored-by` and `processed-by`** — counted maintainer-authored merges as throughput when the real signal is "did a maintainer merge/close this," regardless of who authored it.
4. **No explicit waiting-on signal** — couldn't tell whether open PRs were stuck on maintainer review or author response.

## Design analysis

### 1) Window expansion — `metrics/index.ts:136,150,152`
```ts
// Keep header + last 5000 data rows
if (timeseriesLines.length > 5001) {
  const header = timeseriesLines[0];
  timeseriesLines = [header, ...timeseriesLines.slice(-5000)];
}
```
Three coordinated edits (comment, threshold, slice argument) all moved from `100/101` to `5000/5001`. At one row per cron run this is roughly 50× more history — enough for 7d/30d deltas at most reasonable cadences. The header preservation (`timeseriesLines[0]` always carried over) is correct.

Risks:
- **No size cap** — at 5000 rows the file is bounded but could still grow unbounded if the schema widens. Probably fine for a CSV that's a few KB even at full window.
- **No migration** — existing files at the 100-row trim level just stop being trimmed until they exceed 5000. Acceptable, but worth a one-line note.

### 2) `backlog_age.ts` — new script, +48 lines
Queries the first 100 OPEN issues sorted by `CREATED_AT ASC` (oldest first) and reports `backlog_age_days` as the mean age of those 100 (or 0 if no issues). Math at `:33-41`:
```ts
const totalAge = issues.reduce((acc, issue) => acc + (now - new Date(issue.createdAt).getTime()), 0);
const avgAgeDays = totalAge / issues.length / (1000 * 60 * 60 * 24);
```
Two-decimal rounding via `Math.round(avgAgeDays * 100) / 100` is appropriate for a time-series metric.

Risks:
- **Capped at 100 issues** — for repos with >100 open issues, this measures the age of the *oldest* 100 (since `direction: ASC`), which is a useful "backlog tail" metric but not a true mean. The variable name `backlog_age_days` suggests a global mean but the implementation is the tail-100 mean. Worth either renaming to `backlog_age_oldest_100_days` or paginating to compute a true mean.
- **No pagination handling** — if the GraphQL query is rate-limited or the response is truncated, the script silently reports a smaller-than-expected denominator. A check on `data.issues.pageInfo.hasNextPage` would surface this.

### 3) `bottlenecks.ts` — new script, +74 lines
Samples the last 50 OPEN PRs and their last 10 timeline items (`ISSUE_COMMENT`, `PULL_REQUEST_REVIEW`, `PULL_REQUEST_REVIEW_COMMENT`); buckets each PR by "last actor was the PR author?" → `waiting_on_maintainer`, else `waiting_on_author`. Empty timeline → `waiting_on_maintainer`.

The classification logic at `:46-65` is sound for the simple case but has known holes:
- **"Last actor"** treats a maintainer's "LGTM, please rebase" comment as `waiting_on_author` even if no review is pending — fine.
- **Author-self-comment** (e.g. author pings "any updates?") flips state to `waiting_on_maintainer` even if the maintainer is intentionally waiting on something else. Acceptable noise at the sample-of-50 level.
- **The maintainer-vs-author distinction is implicit** (anyone-not-the-author = maintainer). A drive-by external commenter is treated as a maintainer. Could be tightened with `authorAssociation` filtering (`MEMBER`, `COLLABORATOR`, `OWNER`).

Sample size of 50 is honest about not being a census — the metric names (`prs_waiting_on_maintainer_sample`, `prs_waiting_on_author_sample`) carry the `_sample` suffix, which is good discipline.

Risks:
- **`gh api graphql` shell-out with template-string interpolation** (`-F owner=${GITHUB_OWNER} -F repo=${GITHUB_REPO}`) — if `GITHUB_OWNER` or `GITHUB_REPO` are ever populated from untrusted input, this is a shell injection. They appear to be compile-time constants from `../types.js`, so safe today, but worth converting to `execFileSync` with an args array as defense-in-depth.

### 4) `throughput.ts` — `+56/-16`
Adds `mergedBy { login }` and `closedBy { ... on User { login } ... on Bot { login } }` to the GraphQL query, and the projection at `:43-55` reshapes the per-PR record to include both `association` (author's) and `mergedBy` (processor's). The downstream code (not fully visible in the truncated diff) presumably distinguishes the two for separate maintainer-authored vs. maintainer-processed counters.

The `Bot { login }` arm on `closedBy` is a nice touch — auto-close bots (stale-bot, dependabot rebase) would otherwise be `null` and skew the throughput.

## Risks (cross-cutting)

- **Three new GraphQL queries per cron run** (`backlog_age`, `bottlenecks`, expanded `throughput`) — each costs API budget against the PAT used by the bot. At a typical hourly cadence, 24×3 = 72 extra requests/day, well within rate limits but worth confirming.
- **No tests** — these are bot scripts, not library code; reasonable, but the `bottlenecks.ts` "last actor classification" is the kind of logic that benefits from a unit test against synthetic timeline fixtures.
- **PR title is literally "# Description"** — the PR body uses Markdown headers but the title field accidentally got the first body line. Should be retitled to something like "fix(metrics): expand history window and add backlog/bottleneck visibility" before merge.

## Suggestions

1. Rename `backlog_age_days` → `backlog_age_oldest_100_days` (or paginate for a true mean).
2. Tighten `bottlenecks.ts` author/maintainer distinction via `authorAssociation`.
3. Convert `execSync` to `execFileSync` for shell-injection defense in depth.
4. Add a unit test for the `bottlenecks.ts` last-actor classification.
5. Fix the PR title.

## Verdict

`merge-after-nits` — four real metrics improvements coherently bundled (window expansion is the unblocker for the other three to actually carry signal across deltas), the backlog-age and waiting-on-maintainer signals are useful additions, the throughput refinement (`mergedBy`/`closedBy` separation) is the right semantic fix. Wants the metric rename for honesty, the maintainer-vs-author tightening, and the PR title repair. The shell-injection defense-in-depth and unit tests are nice-to-haves.
