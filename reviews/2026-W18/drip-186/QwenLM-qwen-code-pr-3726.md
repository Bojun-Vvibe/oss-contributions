---
pr: QwenLM/qwen-code#3726
sha: 69a3a42f4044a101b8525ede870dc7ad23950c0b
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# feat(core): add Monitor(...) permission namespace

URL: https://github.com/QwenLM/qwen-code/pull/3726
Files: `packages/core/src/permissions/permission-manager.ts`,
`packages/core/src/permissions/permission-manager.test.ts`,
`packages/core/src/permissions/rule-parser.ts`,
`packages/core/src/tools/monitor.ts`,
`packages/core/src/tools/monitor.test.ts`
Diff: 169+/15-

## Context

Real security-correctness fix. Before this PR, the `monitor` tool
(which runs `tail -f`-style processes) emitted permission rules in
the `Bash(...)` namespace at the existing `monitor.ts:permissionRules`
site, so when a user clicked "Always Allow" for a `monitor` invocation
of `tail -f /var/log/app.log`, the saved rule was `Bash(tail -f *)`.
This had two cross-tool-leak failure modes:

1. **Approval doesn't stick**: a *future* `monitor` invocation of the
   same command would re-prompt — because the saved `Bash(tail -f *)`
   rule was evaluated against `toolName: 'monitor'`, which didn't
   match `'run_shell_command'`. So "Always Allow" silently failed.
2. **Approval over-grants**: simultaneously, a `Bash(tail -f *)` rule
   *did* match `run_shell_command` invocations of the same string,
   meaning the user thought they were approving a monitor and were
   actually granting full shell access to anything matching the
   pattern.

Both directions are wrong, and both are fixed by the same change:
give `monitor` its own `Monitor(...)` permission namespace.

## What's good

- `SHELL_LIKE_TOOLS = new Set(['run_shell_command', 'monitor'])` at
  `permission-manager.ts:36` is the right architectural primitive.
  The two tools share *evaluation semantics* (compound-command
  splitting, AST read-only analysis, virtual file/network op
  matching) but are *distinct permission targets*, and a typed set
  is the cleanest way to express that distinction.
- The cross-tool isolation regression tests at
  `permission-manager.test.ts:1003-1023` ("Monitor approval does NOT
  allow run_shell_command") and `:1025-1041` ("Bash approval does
  NOT allow monitor") are exactly the right shape — they test the
  *exact bug class* the PR is fixing, not just the happy path.
  Future refactors that re-collapse the namespaces will trip these
  immediately.
- Five-axis test coverage at `:973-1042` (allow rule matches,
  deny rule blocks, cross-tool deny, cross-tool deny opposite
  direction, default-allow for read-only). All five are necessary
  and none are redundant.
- `parseRule('Monitor')` and `parseRule('Monitor(tail -f *)')` tests
  at `:140-156` lock the rule-parser side of the change.
- `matchesRule` tests at `:660-680` lock the lower-level matcher
  separately from the PermissionManager — this two-layer test
  design is correct because PermissionManager has its own compound-
  command splitting that could mask matcher-level bugs.
- `buildPermissionRules` test at `:1803-1813` and
  `buildHumanReadableRuleLabel` test at `:1866-1869,1890-1893`
  cover the rule-emission and UI-display sides, ensuring the
  monitor `permissionRules` change at `tools/monitor.ts` actually
  flows through to the user-visible "monitor 'tail -f *' commands"
  label.
- The PR body explicitly notes "206 passed" / "29 passed" /
  `npm run typecheck clean` evidence — the kind of verification
  that should be standard for a permission-system change.
- Follow-up to #3684 review threads 8/10/11 (called out in PR body)
  — this isn't a speculative refactor, it's closing review feedback
  on a recent merged change, with the loop properly closed by tests.

## Nits / follow-ups

- None blocking. The only architectural question worth a future
  follow-up: the `SHELL_LIKE_TOOLS` set will accumulate as new
  shell-spawning tools land (a hypothetical `Watch(...)`,
  `Subprocess(...)`, etc.). Worth promoting it to a registered
  capability on the tool definition itself rather than a
  hardcoded set in `permission-manager.ts`, so new tools opt in
  via their own descriptor instead of via a touch in this file.
  But that's the next refactor, not this PR.

## Verdict

`merge-as-is` — security-correctness fix to a real cross-tool
permission leak, with comprehensive multi-layer test coverage that
locks both the new behaviour and the bug class against regression.
The kind of permission-system change that should land same-day.
