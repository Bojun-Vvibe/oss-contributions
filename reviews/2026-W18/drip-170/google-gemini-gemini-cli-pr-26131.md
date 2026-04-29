# Review: google-gemini/gemini-cli#26131 — Improve Metric Accuracy for Issues, PRs, and Review Distribution

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26131
- **Head SHA:** `7faa50cbaeacb720c903963bc9a28a7ef2a4fa2f`
- **Author:** gemini-cli-robot
- **Diff size:** +115 / -32 across 7 files in `tools/gemini-cli-bot/metrics/scripts/`
- **Verdict:** `merge-after-nits`

## What it does

Three orthogonal fixes bundled into the metrics scripts:

1. **Fix 1000-issue/PR cap.** `open_issues.ts` and `open_prs.ts` previously called `gh issue list --state open --limit 1000 --json number --jq length` and `gh pr list ... --limit 1000 ...`. With ~2400 open issues, the metric was silently capped at 1000. Replaced with a GraphQL `repository.issues(states: OPEN) { totalCount }` (and the analogous `pullRequests` query) which returns the true count.

2. **Stop using shell-interpolated GraphQL queries.** Across 7 scripts, the previous shape was `\`gh api graphql -F owner=${GITHUB_OWNER} -F repo=${GITHUB_REPO} -f query='${query}'\`` — interpolating the query into a shell string. Refactored to `gh api graphql -F owner=$OWNER -F repo=$REPO -f query=@-` with the query passed via `input:` (stdin) and owner/repo as `env`. This eliminates a shell-injection vector if `GITHUB_OWNER` / `GITHUB_REPO` ever contained a quote (currently they're hardcoded constants from `types.js`, so the practical risk was zero, but the new shape is correct).

3. **Add `COLLABORATOR` to maintainer association check.** In `domain_expertise.ts:99` and `review_distribution.ts:43`, the `['MEMBER', 'OWNER']` allowlist is widened to `['MEMBER', 'OWNER', 'COLLABORATOR']`. This includes outside collaborators (people with explicit repo write access who are not org members) in the maintainer-review-distribution metric — they were previously silently excluded.

4. **Surface GraphQL errors instead of swallowing them.** Previously `try { ... } catch { console.log('open_issues,0'); }` masked any failure as "0 issues". New shape: `if (response.errors) throw new Error(response.errors.map((e: any) => e.message).join(', '));` and `catch (err) { process.stderr.write(...); process.exit(1); }`. CI now fails loudly instead of silently reporting zero.

## Specific citations

- `open_issues.ts:14-22` — new GraphQL query `repository.issues(states: OPEN) { totalCount }`. This is the right field — `totalCount` is unbounded and doesn't paginate.
- `open_issues.ts:33-36` — the `process.exit(1)` on failure is the load-bearing observability fix. The old `console.log('open_issues,0')` fallback is a metrics anti-pattern: a downstream dashboard would render "0 open issues" indistinguishably from a healthy zero-backlog state and from a broken metric pipeline.
- `open_prs.ts:14-22` — same shape as open_issues, applied symmetrically. Good.
- `domain_expertise.ts:38-44` and 5 other scripts (`latency.ts`, `review_distribution.ts`, `throughput.ts`, `time_to_first_response.ts`, `user_touches.ts`) — `input: query` + `env: { ...process.env, OWNER, REPO }` + `query=@-` (read from stdin) is the canonical safe shape for `gh api graphql`. Pattern is applied uniformly.
- `domain_expertise.ts:99` and `review_distribution.ts:43` — the `'COLLABORATOR'` addition is a behavioral change to the metric semantics. Anyone reading historical dashboards needs to know the metric definition changed.

## Nits to address before merge

1. **Document the metric semantic change in the PR body.** Adding `COLLABORATOR` to `domain_expertise.ts` and `review_distribution.ts` will cause a one-time bump in maintainer-review counts. If these metrics feed a longitudinal dashboard, annotate the dashboard or add a metric-version bump (e.g. include a `schema_version` column) so post-merge readings aren't compared to pre-merge baselines.
2. **The async iteration over scripts is consistent across 7 files** but `domain_expertise.ts:38` uses `stdio: ['pipe', 'pipe', 'ignore']` while the others use the default. Confirm the asymmetry is intentional (likely: domain_expertise wants to suppress noisy gh stderr) and add a one-line comment explaining why.
3. **`(e: any)` in `response.errors.map((e: any) => e.message)`** — the `any` annotation bypasses TS strict mode. Define a small `interface GraphQLError { message: string }` type once in `types.ts` and reuse across the 7 scripts.
4. **Add a test for `COLLABORATOR` inclusion.** Even a single fixture-based unit test in `domain_expertise.test.ts` asserting that a review by `authorAssociation: 'COLLABORATOR'` now counts (and one by `authorAssociation: 'CONTRIBUTOR'` still does NOT count, to pin the boundary) prevents future "broaden the allowlist further" PRs from accidentally including drive-by external reviewers.
5. **`open_issues.ts` and `open_prs.ts` lost their old fallback.** The previous behavior was "if gh fails, emit 0 and exit 0". The new behavior is "if gh fails, exit 1". For a metrics-collection cron this is the right shape, but if any consumer downstream parses stdout and tolerates an exit-1 process (e.g. a `|| true` in the calling shell pipeline), that consumer needs to be audited.
6. **GraphQL query missing `pageInfo`.** `repository.issues(states: OPEN) { totalCount }` returns total but not paginated nodes — fine for the count metric, but if this same query gets reused elsewhere for a list, the `totalCount`-only shape will be misleading. Add a comment `// totalCount only — do not use for fetching node list`.

## Rationale

Three correct fixes (bypass 1000-cap, harden injection surface, broaden maintainer set) bundled in one PR. The "stop swallowing failures" change is the highest-value: silent zero-fallback in metrics pipelines is a classic source of "we thought the backlog was 0, actually it was on fire for 6 weeks" incidents. Test coverage is the only meaningful gap (no unit tests added for the COLLABORATOR boundary or the totalCount path); approve after the COLLABORATOR-inclusion test and the CHANGELOG/dashboard annotation.