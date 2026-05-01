# google-gemini/gemini-cli #26348 — Metrics updates (window expansion + backlog age)

- URL: https://github.com/google-gemini/gemini-cli/pull/26348
- Head SHA: `d16543017101d24b25cbdb6c900e82b1a2c2041c`
- Author: bot (`app/gemini-cli`)
- Files: `tools/gemini-cli-bot/metrics/index.ts` (+4/-4), `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts` (+59 new)

## Context / problem

Two coupled bot-metrics issues:

1. **Rolling-window too small.** `metrics/index.ts:136,150,152` capped `metrics-timeseries.csv` at 100 data rows. Each run emits ~73 metrics → the window held under 1.5 runs of data → 7-day and 30-day deltas were undefined-by-construction. This is the same window-too-small fix that landed on `5000` in #26350; see drip-249 review.
2. **No backlog-age signal.** Bot tracked open-issue count but not the age distribution of the open backlog. New `backlog_age.ts` script samples the oldest 100 open issues and emits `backlog_age_days,<mean>`.

## Design analysis

### Window expansion (`metrics/index.ts:136,150,152`)

Three coordinated literal swaps `100 → 5000` and `101 → 5001` (the +1 accounting for the header row). Comment update at `:9` and `:18`: "Keep header + last 5000 data rows." Single-call-site change, well-isolated.

This is **the same change as PR #26350's window expansion**, which was already reviewed in drip-249. Both PRs touch the identical lines. If both merge independently they'll conflict on `index.ts`; if one merges first the other becomes a no-op rebase. The PR description acknowledges metrics issues but doesn't flag the overlap with #26350. Worth a coordination note on whichever PR lands second.

### `backlog_age.ts` (`+59` new)

```ts
const query = `
query($owner: String!, $repo: String!) {
  repository(owner: $owner, name: $repo) {
    issues(first: 100, states: OPEN, orderBy: {field: CREATED_AT, direction: ASC}) {
      nodes { createdAt }
    }
  }
}`;
```

Three things to flag against the chosen shape:

1. **Variable name is dishonest.** `backlog_age_days` reads as "average age of the entire open backlog." It is *not*: it is the mean age of the oldest 100 issues (`first: 100, orderBy: CREATED_AT ASC`). For repos with backlog > 100 issues — which is the only case where this metric matters — this is a tail-100 mean, *biased upward by construction*. As the global backlog grows, the metric saturates at the age of the 100th-oldest issue and stops moving even when newer issues are accumulating.
   - Rename to `backlog_age_oldest_100_days` (matches the existing `_sample` honesty pattern from #26350's `bottlenecks.ts` work).
   - Or: also emit `total_open_issues` alongside, so dashboards can divide and detect the saturation regime.

2. **`gh api` shell-injection surface.** The script builds the command with template-string interpolation:
   ```ts
   `gh api graphql -F owner=${GITHUB_OWNER} -F repo=${GITHUB_REPO} -f query='${query}'`
   ```
   `GITHUB_OWNER` and `GITHUB_REPO` come from `'../types.js'` — presumably bot-controlled constants, not env vars — so the practical risk is low. But (a) `execSync` with shell concatenation is the wrong pattern when the argv-form `execFileSync('gh', ['api', 'graphql', '-F', `owner=${...}`, ...])` is one keystroke harder and immune. The query string itself uses a single literal so escaping concerns are limited to the surrounding `''`, but a query containing a `'` would corrupt the shell parse. Defer the safer form to a follow-up if these constants are truly trusted, but flag it.

3. **`stdio: ['ignore', 'pipe', 'ignore']` swallows stderr.** When the GraphQL call fails (rate limit, auth, network), stderr is dropped and the catch block writes `error instanceof Error ? error.message : String(error)` — but `error.message` from `execSync` failure is just `Command failed: gh api graphql ...`, the actual `gh` error body is gone. Switch the third fd to `'pipe'` and include `error.stderr` in the diagnostic string, or use `stdio: 'inherit'` for stderr if the bot host's stderr is captured by the runner anyway.

### Telemetry semantics (minor)

`process.stdout.write('backlog_age_days,0\n')` when `issues.length === 0` is the right "no signal yet" representation (vs. NaN or omitting the line which would break parsers downstream). Good.

`Math.round(avgAgeDays * 100) / 100` rounds to 0.01 days — fine resolution for a metric measured in days.

## Risks

- **Conflict with #26350** (described above) — needs coordination. If the maintainer batches both, the deduplication is trivial; if not, second one to land needs a rebase note.
- **Metric semantic drift** if the rename isn't done — future readers of dashboards will treat `backlog_age_days` as a global mean and draw wrong conclusions about backlog growth.
- **`createdAt` for issues includes both real issues and converted PR-closes-as-issue entries** depending on the repo's convention. The GraphQL `issues(states: OPEN)` connection excludes PRs by default (PRs and issues are separate connections in GraphQL), so this is fine — but worth a one-line comment so the next maintainer doesn't worry.
- **No test coverage** — the script is shelling out to `gh` and parsing GraphQL JSON; a unit test that mocks `execSync` and exercises the empty-backlog and N-issue paths would lock the rounding contract and the empty-state output.

## Suggestions

- Rename `backlog_age_days` → `backlog_age_oldest_100_days` (honesty about what's measured).
- Switch `execSync` to `execFileSync` with argv form, drop the shell.
- Capture stderr (`stdio: ['ignore', 'pipe', 'pipe']`) and include in the catch path's error message.
- Coordinate with #26350 on the `100 → 5000` window change — pick one to land first and rebase the other.
- Add a unit test for empty-backlog (`0`) and N-issue (`compute mean correctly, round to 2dp`) paths.

## Verdict

`merge-after-nits` — correct direction (window-too-small was breaking 7d/30d deltas; backlog-age was a missing signal), but the new metric's variable name overstates what it measures (tail-100 mean ≠ global mean), the `execSync` shell-form is the wrong default for shelling out, and stderr is being swallowed on failure. Coordination with the overlapping #26350 is the main process nit. Each individual change is small and reversible.
