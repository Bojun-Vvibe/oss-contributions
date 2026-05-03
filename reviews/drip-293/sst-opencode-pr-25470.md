# sst/opencode PR #25470 — chore: rm log statement

- **Repo:** sst/opencode
- **PR:** #25470
- **Head SHA:** `13e9b347b59249c2d029c4458be4392408f3bfdb`
- **Author:** rekram1-node
- **Title:** chore: rm log statement
- **Diff size:** +0 / -1 across 1 file
- **Drip:** drip-293

## Files changed

- `packages/opencode/src/permission/index.ts` (-1) — removes `log.info("evaluate", { permission, pattern, ruleset: rulesets.flat() })` from the top of `evaluate()` at line 147.

## Specific observations

- `permission/index.ts:147` — the removed call sat inside `evaluate()`, which is invoked on every permission check. `rulesets.flat()` allocates a new array each call purely to feed the log line, so dropping it is a strictly positive micro-perf change in addition to the noise reduction. Good change.
- The deletion leaves a clean function body (`return evalRule(...)` is now the first statement); no orphaned imports, no comment fragments, no unused variables.
- No tests touched. Permission evaluation already has coverage and removing a `log.info` doesn't change any observable behavior, so that's appropriate.
- Verify `log` is still used elsewhere in the file (it is — search for `log.` in `permission/index.ts` to confirm the import isn't now dead). The diff doesn't show the import block, so a final pre-merge sanity check is worth doing.

## Verdict: `merge-as-is`

Single-line removal of a hot-path debug log that allocated on every call. Mechanically clean, zero behavioral risk, no test churn needed. Confirm the `log` import is still referenced elsewhere in the file and merge.
