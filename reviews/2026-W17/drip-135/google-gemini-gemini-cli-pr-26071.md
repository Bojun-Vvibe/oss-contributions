# google-gemini/gemini-cli #26071 — Fix: 1000-issue metric cap for accurate repository health tracking

- PR: https://github.com/google-gemini/gemini-cli/pull/26071
- Head SHA: `ee347a76fdc4b465d7d17d90edc4542f04f64a38`
- Files changed: 2 (`+36/-10`) — `tools/gemini-cli-bot/metrics/scripts/open_issues.ts`, `tools/gemini-cli-bot/metrics/scripts/open_prs.ts`
- Author: gemini-cli-robot (bot — auto-generated)

## Verdict: merge-after-nits

## Rationale

- **Diagnosis is right.** `open_issues.ts:9` and `open_prs.ts:9` (pre-diff) used `gh issue list --state open --limit 1000 --json number --jq length`, which **caps at 1000** by `gh`'s pagination, so once the repo crossed 1000 open items the metric was permanently `1000` — masking the real backlog. Fix swaps to `gh api "search/issues?q=repo:${repo}+is:issue+is:open" --jq .total_count` which returns the true server-side count without paging through every record. Same fix mirrored in `open_prs.ts` with `is:pr` instead of `is:issue`.
- **`GITHUB_REPOSITORY` env var with `${GITHUB_OWNER}/${GITHUB_REPO}` fallback** at `open_issues.ts:11` / `open_prs.ts:11` is the right shape — Actions environments set `GITHUB_REPOSITORY` automatically, local-run developers fall through to the `types.js` constants. The `||` short-circuit correctly handles both `undefined` and empty-string env.
- **Output format change is intentional and load-bearing.** Old output: plain CSV `open_issues,${count}\n`. New output: structured JSON `{ metric, value, timestamp }\n` written via `process.stdout.write`. The `MetricOutput` type imported from `../types.js` enforces the shape at compile-time. This is a *consumer-breaking change* — any downstream pipeline that was parsing `open_issues,N` will now see JSON. PR body doesn't mention this. Either the consumer was already updated in a prior PR (worth confirming via `gh pr list --search 'MetricOutput'`) or this PR needs a follow-up to update the consumer.
- **Error path now uses `process.stderr.write` with `err.message`** instead of `catch {}` swallow — much more debuggable in CI logs. The `err instanceof Error ? err.message : String(err)` handling correctly covers the "thrown value isn't an Error" cell, and the fallback metric is still emitted as JSON so the downstream pipeline doesn't break on errors. Good shape.
- **`parseInt(count, 10) || 0`** at `:18` correctly handles both the success path and the "gh returned empty/whitespace" cell. The `|| 0` short-circuits `NaN` → `0` matching the previous fallback semantics.

## Nits / follow-ups

- **`open_issues.ts` and `open_prs.ts` are 95% identical.** Both now have the same `GITHUB_REPOSITORY` fallback boilerplate, the same `MetricOutput` JSON-output shape, the same try/catch with stderr+fallback. The two should share a helper like `runGhCountQuery(query: string, metricName: string)` in a sibling `_helpers.ts` — would reduce drift risk where one file gets updated and the other doesn't (cf. drip-132 #26644's "two registry files in lockstep" observation).
- **The `gh api search/issues` endpoint is rate-limited at 30 req/min for unauthenticated and 30 req/min for the search API specifically (separate from the core 5000/hr).** If this metric runs frequently from the same Actions runner, it can hit the search-API rate limit and return 403. Worth either (a) caching the result for ≥2 minutes, (b) catching the specific 403 and emitting a sentinel like `value: -1` so downstream knows it's a rate-limit miss vs. a true zero, or (c) noting in the PR body that this is invoked at a frequency well below the limit.
- **The query `repo:${repo}+is:issue+is:open` works because `gh api` expands the `+` as a literal plus.** This is correct for the GitHub search syntax (which uses `+` as a query separator) but fragile — if `repo` ever contains a character that needs URL-encoding (it can't currently, but someone might extend the env-var fallback to accept e.g. `org/repo with spaces`), the un-encoded plus signs become a footgun. `encodeURIComponent` on each query term would be safer.
- **No test.** A Vitest unit test using `vi.mock('node:child_process')` to stub `execSync` and assert the JSON shape printed to stdout would catch regressions in the output format — exactly the kind of contract the consumer pipeline cares about.
- **Output format change is breaking for any current consumer.** PR body should explicitly call out: "Downstream pipeline at <path> updated to consume JSON in <commit>". If no commit exists, this PR + the consumer update should land together to avoid a metrics-pipeline outage.
