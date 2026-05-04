# google-gemini/gemini-cli #26239 — Backlog Management & Metrics Integrity

- **Head SHA reviewed:** `12c84c67d15b51efc4da3e2174c773b2f1525b48`
- **Size:** +88 / -201 across 9+ files
- **Verdict:** request-changes

## Summary

Two unrelated changes bundled into one PR:

1. Tightens `gemini-scheduled-stale-issue-closer.yml`: creation
   threshold 90 → 60 days, update threshold 10 → 7 days, *and*
   restructures the close-vs-nudge logic so first-pass adds a `Stale`
   label + nudge comment, second pass closes.
2. Strips JSON output and timestamps from a fleet of metrics scripts
   under `tools/gemini-cli-bot/metrics/scripts/`, replacing them with
   `name,value` CSV-style stdout.

These should be two PRs.

## What I checked

- `.github/workflows/gemini-scheduled-stale-issue-closer.yml:50-53` —
  the variable rename `threeMonthsAgo → sixtyDaysAgo` is correct and
  matches the new threshold. Good.
- `.yml:128-162` — the new two-phase logic (nudge → close) is the
  right pattern, but the gap between phases is implicit: an issue is
  closed only when it's *both* stale AND already labeled `Stale`. So
  the *minimum* time from "first detected stale" to "closed" is one
  scheduler-run interval (likely 24 h). A user who responds within
  that interval may have the issue closed on the same run that nudged
  it — because the `hasStaleLabel` check is computed *before* the
  nudge is added. The current code sequence is safe (the nudge
  branch doesn't fall through to the close branch in the same
  iteration), but make this explicit with a comment.
- `.yml:147` — the nudge comment promises closure "in 7 days" but the
  scheduler reads the same `sevenDaysAgo` threshold for *both* the
  initial nudge *and* the close decision. So closure happens 7 days
  after the issue was last updated, not 7 days after the nudge. If the
  issue was already 30 days stale when nudged, it'll close on the
  *next* run. The comment is misleading; either change the comment to
  "soon" or track the nudge date separately.
- `tools/gemini-cli-bot/metrics/scripts/domain_expertise.ts:99-101` —
  adds `'COLLABORATOR'` to the maintainer reviewer set. Reasonable
  expansion, but coupled with the output-format change it's hard to
  isolate.
- Same file lines 138-141 — drops the JSON wrapper for `name,value`
  CSV. This is a **breaking change** for anything downstream that
  parses these scripts' stdout. Multiple scripts are affected
  (`latency.ts`, `throughput.ts`, etc.). The PR body doesn't mention
  who consumes this output or whether the consumer is updated in
  lock-step. Until that's clear, this is request-changes.
- `latency.ts:96-105` — drops 4+ fields per metric (`metric`,
  `timestamp`, `details`). If anything stores these for trend
  analysis, you lose the dimension columns.

## Risk

Medium-high on the metrics output change (silent breakage for
downstream consumers); low on the workflow tweak.

## Recommendation

Request changes:

1. **Split the PR**: backlog automation and metrics format are
   independent concerns and need to be reviewable / revertible
   separately.
2. For the workflow: add a `nudgedAt`-timestamp tracker (e.g.,
   embed in the comment or a separate label) so the "we'll close
   in 7 days" promise is honored literally.
3. For the metrics: state explicitly which downstream consumes the
   output and update it in the same PR (or a paired one), or keep
   JSON as a `--json` flag and default to CSV.
4. Consider keeping `timestamp` even in CSV mode — losing it makes
   trend storage impossible without an external clock.
