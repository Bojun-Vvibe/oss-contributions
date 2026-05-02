---
repo: sst/opencode
pr: 25470
head_sha: 13e9b347b59249c2d029c4458be4392408f3bfdb
title: "chore: rm log statement"
verdict: merge-as-is
reviewed_at: 2026-05-03
---

# Review: sst/opencode#25470 — `chore: rm log statement`

**Head SHA:** `13e9b347b59249c2d029c4458be4392408f3bfdb`
**Stat:** +0 / −1 across 1 file. Closes #17218.

## What it changes

Single-line removal in `packages/opencode/src/permission/index.ts:147`:

```diff
 export function evaluate(permission: string, pattern: string, ...rulesets: Ruleset[]): Rule {
-  log.info("evaluate", { permission, pattern, ruleset: rulesets.flat() })
   return evalRule(permission, pattern, ...rulesets)
 }
```

The removed `log.info` was firing on every permission evaluation — i.e. on
every tool invocation, every config refresh, every UI re-render that asks
"can I do X?". Issue #17218 reports that this was producing a torrent of
`evaluate` lines in `info` logs, drowning out signal during real triage.

## Assessment

- Correctness: trivial. Removing a logger call cannot change behavior of
  `evalRule(...)`, which is the only thing the function returns.
- Telemetry impact: the line was at `info` level (not `debug`), so users on
  default log levels were paying the cost. There is no equivalent debug-level
  log being added in its place, but for a hot-path predicate that's the right
  call — if someone needs to debug rule evaluation they'll set the whole
  `permission` module to `debug` and instrument deliberately.
- Scope: the diff touches nothing else; no test changes needed (pure
  log-removal, no observable behavior under test).

## Nits (none blocking)

None. If maintainers wanted a paranoid version they could downgrade to
`log.debug(...)` instead of deleting, but the spammy-by-default complaint in
#17218 makes outright removal the cleaner fix.

## Verdict

**`merge-as-is`** — minimal, targeted, fixes a real noise complaint, no
follow-up required.
