# google-gemini/gemini-cli PR #26348 — Metrics updates

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26348
- **Author:** app/gemini-cli (bot)
- **Head SHA:** `d1654301` (full: `d16543017101d24b25cbdb6c900e82b1a2c2041c`)
- **State:** MERGED
- **Files touched:**
  - `tools/gemini-cli-bot/metrics/index.ts` (+4 / -4)
  - `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts` (+59 / -0, new)

## Verdict

**merge-after-nits**

## Specific refs

- `tools/gemini-cli-bot/metrics/index.ts:133-152` — rolling-window cap raised from `100` data rows to `5000`. Comment + magic-number are updated together (lines 133, 146, 148, 150). The rationale is sound: with ~73 metrics per run, a 100-row cap purged ~99% of history every run and broke 7-day/30-day deltas. 5000 rows is ~68 runs of headroom, which is fine if runs are daily but only ~2 months at hourly cadence — worth surfacing the cadence/retention assumption either as a constant (`const TIMESERIES_RETENTION_ROWS = 5000`) or as a comment describing the expected cadence.
- `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts:32-39` (`execSync(\`gh api graphql -F owner=${GITHUB_OWNER} -F repo=${GITHUB_REPO} -f query='${query}'\`)`) — string interpolation into a shell command. `GITHUB_OWNER`/`GITHUB_REPO` come from `../types.js` (presumably a constant module), so this is not user-input injection in practice, but it's still a `gh` invocation that could fail surprisingly if either constant ever picks up a value with quotes. Prefer `execFileSync('gh', ['api', 'graphql', '-F', `owner=${GITHUB_OWNER}`, ...])` to avoid the shell layer entirely.
- `backlog_age.ts:48` — output format `backlog_age_days,${Math.round(avgAgeDays * 100) / 100}\n`. CSV emission with no header here is intentional (it gets concatenated into `metrics-timeseries.csv` upstream) but worth a comment so the contract isn't accidentally broken.

## Rationale

Two small, defensible changes to a metrics script. The retention bump is a real bug fix with a clear before/after delta. The new `backlog_age.ts` is straightforward — 100 oldest open issues, average age in days, single CSV row. The only review-worthy items are nits: pull the retention magic number into a named constant, switch the `execSync` to `execFileSync` to avoid a latent shell-quoting cliff, and add a one-line comment on the output contract. None of these block the metrics value the PR delivers; happy to merge after any one of them lands.

