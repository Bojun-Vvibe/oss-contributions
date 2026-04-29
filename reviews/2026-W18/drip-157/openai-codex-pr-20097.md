# openai/codex PR #20097 — Refine Codex issue digest summaries

- **PR**: https://github.com/openai/codex/pull/20097
- **Author**: etraut-openai
- **Merged**: 2026-04-28T23:53:59Z
- **Head SHA**: `4f898aeb6110`
- **Size**: +133/-31 across 3 files (skill prose + collector script + tests)
- **Verdict**: `merge-after-nits`

## Context

The `.codex/skills/codex-issue-digest/` skill drives a daily-ish issue
triage report on `openai/codex`. The previous SKILL prose mandated a
two-section report (`## Summary` then `## Details`) where Summary was a
bullet pile of headlines and stats and Details was always-rendered. The
collector script that feeds the skill (`collect_issue_digest.py`) used
attention-marker thresholds of 10/20 human interactions for `🔥`/`🔥🔥`
over a 24h window, and pulled GitHub search results in default
`best-match` order. Two operator-side complaints were driving change:
(a) the Summary section buried the lede under repeated counts, and
(b) `🔥` markers fired too rarely on a slow-traffic 24 h window to be
useful as an attention signal.

## What changed

Three coordinated edits:

1. **Mode split + headline-first Summary** (`SKILL.md:32-93`). Three
   output modes are now defined explicitly: `default` (Summary only,
   ending with a one-liner offering details), `details-upfront` (when
   the user asks for table/details/full digest), and `follow-up details`
   (regenerate Details from the cached collector JSON if still fresh,
   else rerun the collector). The Summary contract is rewritten: the
   first nonblank line under `## Summary` MUST be a single-line headline
   or judgment, not a bullet. Quiet days have a canonical literal:
   `No major issues reported by users.` Active days lead with a count or
   theme line (e.g. `Two issues are being surfaced by users:`) and then
   list only the surfacing rows ordered by importance, each prefixed with
   the row's `attention_marker` if present. Crucially, two example
   markdown blocks are inlined at `SKILL.md:55-77` so the skill consumer
   has a literal template to anchor against.

2. **Threshold halving** (`scripts/collect_issue_digest.py:14-18`):

   ```diff
   -ONE_ATTENTION_INTERACTION_THRESHOLD = 10
   -TWO_ATTENTION_INTERACTION_THRESHOLD = 20
   +ONE_ATTENTION_INTERACTION_THRESHOLD = 5
   +TWO_ATTENTION_INTERACTION_THRESHOLD = 10
   ```

   And `SCRIPT_VERSION` bumps `2 → 4`. The new baseline reads "5 human
   interactions in 24 h to earn a single fire, 10 to earn a double" —
   a 2× sensitivity bump. Window-scaling math is unchanged (still
   linear+ceil), so a one-week digest now uses 35/70 instead of 70/140.

3. **Search ordering + per-query cap fix** (`scripts/collect_issue_digest.py:307-340`).
   The GitHub search call now passes `sort=updated` + `order=desc`
   explicitly, and the per-query pagination accumulator is renamed from
   `len(numbers) >= limit` to a per-query `seen_for_query >= limit`
   counter. The original logic short-circuited when the *combined*
   collected number across all queries hit `limit`, which under
   multi-label invocations starved later queries. The new counter is
   reset per query, which is the correct shape for "give me up to N
   results from each label."

## Why `merge-after-nits`

The headline-first redesign is genuinely better for the operator surface
— a daily digest that opens with `No major issues reported by users.`
on a quiet day is exactly the right ergonomic — and the test addition
at `test_collect_issue_digest.py:54-130` covers the new
`sort=updated`/`order=desc` request shape (`test_search_issue_numbers_requests_updated_sort`).
Three things would have warranted a one-day pause before merge:

- **Threshold halving without a backtest.** Cutting the `🔥` cutoff
  from 10 to 5 doubles the rate at which the marker fires, but the PR
  ships no historical replay showing what the past N digests would have
  looked like under the new bounds. If `🔥` was tuned conservatively
  on purpose because the team reads the marker as "drop everything,"
  doubling the hit rate erodes the marker's signal value. A two-line
  comment near the constant — *"baseline calibrated on $WINDOW; halved
  on $DATE because feedback was that single-fire was too rare on quiet
  weekdays"* — would let the next maintainer revert principle-fully.

- **`sort=updated` interacts with the existing reaction-handling note.**
  The unchanged Reaction Handling section at `SKILL.md:91-99` warns
  that GitHub issue search is "still seeded by issue `updated_at`, so
  a purely reaction-driven uptick may be missed." Adding explicit
  `sort=updated` makes that caveat *more* load-bearing, not less, since
  the sort guarantees the seeding behavior the caveat already worried
  about. Worth either deleting the caveat (because explicit sort makes
  it deterministic) or strengthening it (because `order=desc` means a
  paginated cap *will* deterministically drop reaction-only issues).

- **The per-query `seen_for_query` reset is buried in a paginator
  rewrite.** The diff at `collect_issue_digest.py:307-340` mixes two
  things — adding the `sort`/`order` params and fixing the per-query
  pagination cap — in a single hunk. Both are correct fixes but a
  reader chasing a regression in cap behavior won't immediately spot
  the second one. Commit-splitting would have helped; not worth
  blocking on at this point.

## What I learned

When tightening or relaxing an *attention threshold* (a number that
controls how often a UI affordance fires), the defensible diff is
"old=N, new=M, observed-ratio-on-historical-corpus shifts from X to Y."
The PR has that information privately (etraut-openai presumably tuned
this against real digests) but it doesn't end up in the code or the PR
body, so the next person to touch the constant has to re-derive the
calibration. Embedding the calibration assumption next to the constant
costs one comment line and saves the next bump from being unprincipled
— same lesson as the test-timeout review above, just applied to
attention-marker math.
