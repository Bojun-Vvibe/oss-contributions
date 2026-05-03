# google-gemini/gemini-cli PR #26344 — Bot: Metrics Integrity & Triage Hygiene

- **Repo:** google-gemini/gemini-cli
- **PR:** #26344
- **Head SHA:** `4810e7794b4a3a088b24d367603b67134b238d10`
- **Author:** gemini-cli-robot
- **Title:** Bot: Metrics Integrity & Triage Hygiene
- **Diff size:** +177 / -87 across 5 files
- **Drip:** drip-292

## Files changed

- `.github/workflows/gemini-scheduled-stale-issue-closer.yml` (+102/-76) — replaces the single-pass "find old, close" loop with a two-phase pipeline (Phase 1: mark stale at 90d; Phase 2: close at +14d).
- `.github/workflows/stale.yml` (+4/-8) — small policy tweak.
- `.github/workflows/unassign-inactive-assignees.yml` (+3/-0) — minor adjustment.
- `tools/gemini-cli-bot/metrics/index.ts` (+3/-3) — wires the new backlog metric.
- `tools/gemini-cli-bot/metrics/scripts/backlog_age.ts` (+65/-0) — new script computing median + P90 age of open issues/PRs.

## Specific observations

- `gemini-scheduled-stale-issue-closer.yml:48-54` — `ninetyDaysAgo` (was `threeMonthsAgo`) and `fourteenDaysAgo` (was `tenDaysAgo`). Switching from month-arithmetic to day-arithmetic is the right call (months have variable length and `setMonth` has end-of-month rollover surprises). Good.
- `gemini-scheduled-stale-issue-closer.yml:59-83` — Phase 1 query `is:open -label:"${staleLabel}" updated:<${ninetyDaysAgo}` correctly excludes already-stale issues so Phase 1 doesn't re-mark. Phase 2 inversely scopes to `label:"${staleLabel}" updated:<${fourteenDaysAgo}`. Clean separation.
- `gemini-scheduled-stale-issue-closer.yml:85-100` (`hasHumanActivity`) — paginates `listComments` with `per_page: 100` per call. For an issue with hundreds of comments this is expensive on every Phase 1 candidate. Consider passing `since: sinceDate.toISOString()` to `listComments` to short-circuit; the GitHub REST API supports it and it would dramatically cut request volume.
- `gemini-scheduled-stale-issue-closer.yml:108-115` — exempt-label list now includes `pinned` and `security` (in addition to `maintainer*`, `help wanted`, `🗓️ Public Roadmap`). `security` exemption is critical and was missing before; the omission was a real gap.
- `gemini-scheduled-stale-issue-closer.yml:103-104, 144-145` — `if (staleCount >= 100) break` and `if (closeCount >= 100) break` per-run caps. Good defence against runaway runs, but the cap is silent — a `core.warning('Phase N hit per-run cap of 100')` would make it visible when the backlog is too large for one cycle.
- `gemini-scheduled-stale-issue-closer.yml:158-170` — when Phase 2 finds human activity since the stale label was applied, it removes the `Stale` label rather than closing. Right behavior. Verify the label name passed to `removeLabel` (`name: staleLabel`) matches the exact case used elsewhere — workflow uses `'Stale'` consistently in this hunk so should be fine.
- The bot reuses `gemini-cli-robot` for all writes — confirm the repo's branch protections and stale-comment templates match the new wording (`"...will be closed in 14 days..."`) and that the prior single-stage messaging isn't still referenced elsewhere.
- `backlog_age.ts:1-65` (not visible in this diff slice) — the median/P90 metric directly addresses the survivorship-bias point. Worth ensuring it's actually scheduled to run (check `metrics/index.ts:+3/-3` wires it into the runner).

## Verdict: `merge-after-nits`

The two-phase stale model is a strict improvement over the single-pass close-after-90-days behaviour, and adding `security`/`pinned` exemptions closes a real footgun. Three asks: (1) pass `since:` to `listComments` to cut API spend, (2) emit a `core.warning` when the per-run 100-item cap trips, and (3) confirm the `backlog_age.ts` metric is actually wired into the scheduled runner via `metrics/index.ts`.
