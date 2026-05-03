# Review: google-gemini/gemini-cli PR #26302

- **Title:** Proactive Improvement: Backlog Health & Stale Policy Optimization
- **Author:** gemini-cli-robot (bot-authored)
- **Head SHA:** `5328faff2fcc41424e68e817e8a745d0e4e3c11e`
- **Verdict:** request-changes

## Summary

Bot-authored PR that splits the existing `stale.yml` workflow into two
jobs (`stale-issues` and `stale-prs`), broadens the exempt-label list,
adds `operations-per-run` caps, and introduces two new TS metric
scripts (`backlog_health.ts`, `stale_ratio.ts`) that query GraphQL via
`execSync('gh api graphql ...')`. The split into two jobs and the
explicit `only-issues` / `only-prs` flags are reasonable. The metric
scripts have a security issue.

## Specific-line comments

- `tools/gemini-cli-bot/metrics/scripts/backlog_health.ts:34-36` —
  ```ts
  const output = execSync(
    `gh api graphql -F owner=${GITHUB_OWNER} -F repo=${GITHUB_REPO} ${cursor ? `-F cursor=${cursor}` : ''} -f query='${query}'`,
    { encoding: 'utf-8' },
  );
  ```
  This is a shell command built by string interpolation, with `cursor`
  coming from the GitHub API. GitHub cursors are base64-ish and
  *should* be safe, but there is no validation here, and `query` is
  also interpolated raw via single quotes — if `query` ever contains a
  literal `'`, the command breaks (or worse, splits). Use `execFile`
  with an argv array, or at least pass `query` over stdin via
  `child_process.spawnSync('gh', ['api', 'graphql', '-F', ...], { input: query })`.
- `tools/gemini-cli-bot/metrics/scripts/stale_ratio.ts:16-23` — same
  pattern. Additionally, this template literal embeds
  `${GITHUB_OWNER}/${GITHUB_REPO}` directly into the GraphQL `search`
  query string, which means a poisoned env var would inject GraphQL
  syntax. Bind via GraphQL variables instead of string-templating.
- `tools/gemini-cli-bot/metrics/scripts/stale_ratio.ts:18` — the
  GraphQL `search` syntax `label:stale OR label:Stale` is **not** a
  valid combination of GitHub search qualifiers; `OR` is not supported
  for `label:` in code search and only partially in issue search
  (recent change). This will likely match only the literal token
  `label:stale` and discard the rest. Test with both casings against
  the live API before relying on the metric.
- `.github/workflows/stale.yml:29-47` — splitting into `stale-issues`
  and `stale-prs` is fine, but the original workflow had
  `concurrency: ${{ github.workflow }}-stale, cancel-in-progress: true`
  — the new version drops this entirely. The two jobs can now overlap
  with each other or with a manual `workflow_dispatch` run. Restore a
  concurrency group, scoped per job.
- `.github/workflows/stale.yml:41` and `:62` — exempt labels
  broadened to include `needs-triage,waiting-on-maintainer,status: blocked`.
  The `status: blocked` entry contains a space; verify the
  `actions/stale` action accepts label names with embedded spaces in a
  comma-separated list (some YAMLs require quoting each entry).
- `.github/workflows/stale.yml` (overall) — diff also drops the
  matrix-runner pattern in favor of plain `runs-on: ubuntu-latest`.
  Fine, but if any organizational policy required the matrix shape,
  this regresses it.

## Risks / nits

- No tests for the new metric scripts, so the broken `OR` qualifier
  and the shell-injection-shaped surface won't be caught in CI.
- Bot-authored PR with no human revisions visible — the gemini-cli-bot
  meta-workflow this is *part of* (#26303) is exactly the surface that
  should have caught the shell-quoting issue under its new "Defensive
  Scripting & Resilience" rule. Worth flagging that the bot violated
  its own freshly-added guardrail.

## Verdict justification

Workflow split is mostly fine, but the metric scripts ship a real
shell/GraphQL-injection-shaped pattern, drop the original concurrency
guard, and use a search qualifier (`OR` on `label:`) that does not
behave the way the script assumes. **request-changes** —
parameterize the `gh api graphql` calls, fix the search query, and
restore a concurrency group.
