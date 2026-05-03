# Review: google-gemini/gemini-cli #26293 — Metrics Standardization & Quality Fixes

- PR: https://github.com/google-gemini/gemini-cli/pull/26293
- Head SHA: `868913e446bbf2ec5ea64c19ca11be772b1bece3`
- Author: app/gemini-cli (bot)
- Size: +85 / -251

## Summary

Net-deletion cleanup of the `tools/gemini-cli-bot/metrics/` scripts
after a previous CSV-format migration. Removes the `MetricOutput`
TypedDict and all 6 `@typescript-eslint/no-unused-vars` errors that
followed; deletes redundant duplicate `@license` JSDoc tags; migrates
`open_issues.ts` and `open_prs.ts` from the legacy `gh issue list`/
`gh pr list` shell pipelines to `gh api graphql -F` with named
variables; standardises all `execSync` calls to
`stdio: ['ignore', 'pipe', 'ignore']` so the metric scripts can no
longer accidentally inherit the parent's stdin/stderr.

## Specific citations

- `tools/gemini-cli-bot/metrics/index.ts:42-57` — the
  `processOutputLine` function loses ~30 lines of try/JSON-parse
  logic. The CSV path was already the only real path; the JSON
  fallback existed because earlier metric scripts emitted JSON. With
  every emitter now emitting `name,value\n` strings (see e.g.
  `latency.ts:223-228`), the fallback is dead code and removing it is
  the correct simplification.
- `tools/gemini-cli-bot/metrics/scripts/domain_expertise.ts:60-77` —
  the `authorCache` `Map<string,string>` was previously declared
  *inside* the per-PR loop, which silently disabled the cache (a
  fresh map per PR is no cache). The PR hoists it above the loop, so
  repeated `git log -- <path>` calls across PRs that touch the same
  files now hit the cache. This is a real perf fix masquerading as a
  cleanup PR — `git log --` per file across hundreds of PRs is the
  hot path of this script.
- `tools/gemini-cli-bot/metrics/scripts/domain_expertise.ts:131` —
  widens the maintainer-association set from `['MEMBER', 'OWNER']` to
  `['MEMBER', 'OWNER', 'COLLABORATOR']`. This is a *semantic* change
  not flagged in the PR title — adding `COLLABORATOR` will move the
  `domain_expertise` numeric value for any repo that grants
  collaborator-level access to non-org members. Worth surfacing.
- `tools/gemini-cli-bot/metrics/scripts/latency.ts:223-228` — six
  `JSON.stringify` blocks collapse to six `process.stdout.write`
  CSV-line writes. Unambiguous improvement; one less roundtrip
  through a typed wrapper for output that is structurally just
  `name,value`.
- `tools/gemini-cli-bot/metrics/scripts/open_issues.ts:5-21` and
  `open_prs.ts:5-21` — replace the `--limit 1000 --json number --jq
  length` pattern with a `totalCount` GraphQL query. Correctness win:
  the old pattern silently caps at 1000, so any repo with >1000 open
  issues was reporting 1000. The GraphQL `totalCount` is unbounded.
  Also a nice security improvement — `gh api graphql -F owner=$O -F
  repo=$R -f query='...'` parameterises the inputs instead of string-
  interpolating them into a shell command line.
- `tools/gemini-cli-bot/metrics/types.ts` — full removal of the
  `MetricOutput` interface (the file is now 7 lines lighter). Safe
  given no remaining references.

## Verdict

**merge-as-is**

## Rationale

Pure-cleanup PRs in tooling directories are usually a yes. This one
is unusually high quality: the dead-code removal is sound, the
caching fix is a real bug, the GraphQL migration removes a silent
1000-row truncation, and the stdio hardening is good hygiene. The
diff is net-negative (-166 lines) which is the right direction for a
post-migration cleanup.

The one item that should be called out — even if not blocking — is
the `COLLABORATOR` widening at `domain_expertise.ts:131`. That
genuinely changes the metric's semantics. A separate PR with a clear
title (`metrics(domain_expertise): include COLLABORATOR association
in maintainer set`) would have been cleaner so the timeseries
discontinuity has a discoverable cause when someone investigates the
chart later. Since it's bundled here, at minimum the PR description
should mention it.

The CSV emission lines (e.g. `latency.ts:223-228`) hard-code six
`process.stdout.write` statements with `Math.round(... * 100) / 100`
repeated six times. A small `csvLine(name, value)` helper would
remove the repetition and centralise the rounding policy if it ever
needs to change. Optional polish, not a merge blocker.

The PR body claims `npm run lint` passes — I'd want CI confirmation
before merging, but the diff itself looks lint-clean.

No tests added. These are bot scripts that emit numeric output to
stdout; I'd love a snapshot test fixture per script (mocked `gh api`
output → expected CSV line) but that's a meta-tooling investment
that's well beyond this cleanup PR's scope.
