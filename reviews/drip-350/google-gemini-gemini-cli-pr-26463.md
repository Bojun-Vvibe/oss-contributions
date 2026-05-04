# google-gemini/gemini-cli#26463 — feat(bot): add actions spend metric script

- **URL**: https://github.com/google-gemini/gemini-cli/pull/26463
- **Head SHA**: `b58f921d69ff`
- **Diffstat**: +72 / -0
- **Verdict**: `merge-after-nits`

## Summary

Adds `tools/gemini-cli-bot/metrics/scripts/actions_spend.ts` that fetches the last 1000 GitHub Actions workflow runs via `gh run list`, filters to a rolling 7-day window, and emits per-workflow plus overall minute totals in the bot's `MetricOutput` CSV-ish format. Also adds a one-line entry in `tools/gemini-cli-bot/brain/metrics.md` flagging cost savings as a lowest-priority signal.

## Findings

- `tools/gemini-cli-bot/metrics/scripts/actions_spend.ts:9-32` — `getWorkflowMinutes()` calls `execFileSync('gh', ['run', 'list', '--limit', '1000', '--json', 'workflowName,startedAt,updatedAt'])`. Two real concerns:
  1. **Wall-clock != billed minutes.** `(updatedAt - startedAt)` is the wall-clock duration of the run, not the sum of billable job minutes. A run with 5 parallel jobs of 10 minutes each will show as ~10 min here but bills as ~50 min. For an actual "spend" metric, `gh api /repos/{owner}/{repo}/actions/runs/{run_id}/timing` returns the authoritative `billable.{UBUNTU,MACOS,WINDOWS}.total_ms` per run — but at 1000-run scale that's 1000 extra API calls. At minimum the metric name should be `actions_wallclock_minutes` and the `metrics.md` entry should note the caveat, or the script should fall back to per-OS multipliers (macOS 10x, Windows 2x) the way GitHub bills.
  2. **`--limit 1000` is a hard ceiling.** For high-velocity repos this will silently truncate the 7-day window and under-report. Either paginate until the oldest seen run is older than `sevenDaysAgo`, or assert and warn when `runs.length === 1000 && oldestRun >= sevenDaysAgo`.
- `actions_spend.ts:24` — `if (durationMinutes >= 0)` silently swallows any clock-skew case where `updatedAt < startedAt`. Should at least `console.warn` so operators notice if the GH API ever returns weird ordering.
- `actions_spend.ts:43-47` — `name.replace(/[^a-zA-Z0-9]/g, '_').toLowerCase()` can collide (e.g. `Lint / Build` and `Lint-Build` both become `lint___build` vs `lint_build`). Low-risk in practice but a comment acknowledging it would be honest.
- `actions_spend.ts:51-55` — error path writes to stderr without a newline (`error.message` may or may not have one). Minor cosmetic.
- `tools/gemini-cli-bot/brain/metrics.md:50` — added line: `**Cost Savings (Lowest Priority)**: Monitor actions_spend_minutes…`. Note the metric name in the doc is `actions_spend_minutes` but the script emits `actions_spend_overall_minutes` and `actions_spend_workflow_<name>_minutes`. Doc should match emitted names.

## Recommendation

Useful signal, but the wall-clock-vs-billable conflation is a meaningful accuracy bug for anything labeled "spend". Rename to wall-clock or switch to `/timing` API (with paginated capping), align the doc to actual metric names, fix the silent 1000-row truncation, then ship.
