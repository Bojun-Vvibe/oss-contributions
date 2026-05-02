# google-gemini/gemini-cli PR #26350 — Fix metrics history retention & accuracy

- Head SHA: `6ec068c2ab3cc324ed1c9cf67330d5c1ed13ddd5`
- URL: https://github.com/google-gemini/gemini-cli/pull/26350
- Size: +182 / -20, 4 files
  (`tools/gemini-cli-bot/metrics/index.ts`,
  `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts` *(new)*,
  `tools/gemini-cli-bot/metrics/scripts/bottlenecks.ts` *(new)*,
  `tools/gemini-cli-bot/metrics/scripts/throughput.ts`)
- Verdict: **request-changes**

## What changes

Four operator-side changes to the repo's internal metrics bot:

1. **Rolling window 100 → 5000 lines** (`metrics/index.ts:136-152`).
   Comment was already misleading at 100; new window is 5000 plus
   header.
2. **Restore `backlog_age` metric** (`scripts/backlog_age.ts`, new) —
   GraphQL fetch of the 100 oldest open issues, average age in days,
   emit as a single CSV row.
3. **Restore `bottlenecks` metric** (`scripts/bottlenecks.ts`, new) —
   sample last 50 open PRs, classify each as
   `waiting_on_maintainer` vs `waiting_on_author` based on whether the
   most recent timeline item author matches the PR author.
4. **Throughput refactor** (`scripts/throughput.ts`) — rename
   `pr_maintainers` → `pr_maintainers_authored`, add
   `pr_maintainers_merges` (PRs *closed* by a non-bot account, used as
   a proxy for "processed by maintainers"). Equivalent split for
   issues.

## What looks good

- The comment / constant drift fix in `metrics/index.ts:136-152` is
  long overdue; "100 rows = ~1.5 runs" is a real signal-to-noise
  problem for any 7-day delta.
- The "authored vs processed" distinction in `throughput.ts:80-95`
  *is* the right framing for maintainer-velocity metrics. Burnout
  signals will show up earlier in `_merges` than in `_authored`.
- Both new scripts are self-contained CLI tools (single `try { ...
  execSync } catch { process.exit(1) }`), matching the convention
  the existing scripts in this directory follow.

## Blocking issues

1. **Shell injection in both new scripts.** `backlog_age.ts:24` and
   `bottlenecks.ts:33` interpolate `GITHUB_OWNER` and `GITHUB_REPO`
   straight into a `gh api graphql -F owner=${GITHUB_OWNER} -F
   repo=${GITHUB_REPO} -f query='${query}'` shell command via
   `execSync(string)`. If `GITHUB_OWNER` or `GITHUB_REPO` are ever
   sourced from env / repo metadata that an external contributor can
   influence (workflow_dispatch input, fork PR title, anything), this
   becomes a command-injection. Even in the "trusted constant" case
   today, the *existing* sibling scripts in the same directory
   (`throughput.ts`) appear to use the same pattern — but that's an
   argument to fix the pattern centrally, not to repeat it.
   - Concrete fix: switch to `execFileSync('gh', ['api', 'graphql',
     '-F', \`owner=${GITHUB_OWNER}\`, '-F', \`repo=${GITHUB_REPO}\`,
     '-f', \`query=${query}\`])`. Same wire format, no shell.
2. **Single-quote escape bug.** Both scripts wrap the GraphQL query
   in single quotes (`-f query='${query}'`). The query body
   (`bottlenecks.ts:36-50`) currently contains no single quotes, but
   the moment someone edits the inline GraphQL to include one
   (`... where: {field: "...'..."}`), the shell will silently truncate
   the query and `gh api graphql` will return a parse error from
   *partial* input — extremely hard to debug. Same `execFileSync`
   refactor fixes this.
3. **`bottlenecks.ts` waiting-on-author logic is inverted.** Lines
   65-69:
   ```
   if (lastActor === author) {
     waitingOnMaintainer++;
   } else {
     waitingOnAuthor++;
   }
   ```
   If the *PR author* posted the most recent timeline item, the PR is
   waiting on a *maintainer review* — that part is correct. But the
   `else` branch labels "anyone-not-author posted last" as
   `waitingOnAuthor`. That's the right intent (maintainer responded,
   ball is back in author's court), **except** when the
   "anyone-not-author" is *another bot* (CI, Renovate, dependabot,
   github-actions). A CI status comment from `github-actions[bot]` will
   currently increment `waitingOnAuthor` even though the PR is still
   waiting on a maintainer review. Filter `lastActor` against the same
   `bot` heuristic the throughput script uses
   (`!login.toLowerCase().includes('bot')`).

## Nits

1. `throughput.ts:99-105`: "we don't have the list of maintainers
   here, but we can assume if they merged it, they are likely a
   maintainer" — fine as a v1, but please at least restrict to
   `authorAssociation in ('OWNER', 'MEMBER', 'COLLABORATOR')` on the
   PR; otherwise a community contributor who got merge access via a
   workflow auto-merge will land in `pr_maintainers_merges`.
2. `bottlenecks.ts:38` queries `pullRequests(states: OPEN, last: 50)`
   without a `timelineItems` page-cursor. PRs with > 10 timeline items
   will give a wrong "last actor" because `timelineItems(last: 10)` is
   the *last 10 of any kind* — a PR with 11 review comments and a
   final merge-conflict notice will misclassify. Consider
   `last: 1` and let the rest of the timeline drop, or paginate.
3. `metrics/index.ts:152` — the new 5001 cap is hard-coded twice
   (line 149 and 152). Pull it out as a `const ROLLING_WINDOW = 5000;`
   so the constants stay in sync.

## Risk

Medium. Items 1 & 2 above are real security / robustness defects in
files that are about to land in a tools directory; even though it's
internal automation, the injection vectors are easy to fix and the
fix doesn't change observable behavior. Item 3 silently produces
wrong dashboards, which is the *worst* kind of metrics bug because no
one will notice until a quarterly review.

I'd be happy to flip this to `merge-after-nits` once the
`execFileSync` refactor and the bot-author filter land. The retention
and throughput-split changes are independently good and worth
shipping.
