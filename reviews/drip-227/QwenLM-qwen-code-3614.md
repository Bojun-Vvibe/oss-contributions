---
pr: QwenLM/qwen-code#3614
title: "test(arena): cover select dialog key actions"
head_sha: 1000f568da8dbeba2b33bd83bd7ea768fd735b83
verdict: merge-as-is
drip: drip-227
---

## Context

`ArenaSelectDialog` is the TUI surface that lets the user pick which
arena agent's diff to apply at the end of a multi-agent run. Before
this PR, the test file had a single happy-path render test for the
preview/diff toggle. Four user-facing key actions — Escape, `x`,
Enter-on-success, Enter-on-failure — were entirely uncovered, even
though three of them mutate state (apply changes / cleanup runtime /
close dialog) in ways that are surprisingly subtle: e.g. `x` must call
`cleanupArenaRuntime(true)` (the `true` is the load-bearing arg
meaning "discard worktrees") *and* close the dialog, while Escape must
do *neither* mutation but still close.

## Diff walkthrough

The PR refactors the existing test by extracting a shared
`createDialogHarness(agents = [createAgentResult()])` factory at
`ArenaSelectDialog.test.tsx:122-160` and a `createAgentResult({
agentId, status })` helper at `:162-205`. The original test (lines
20-89 of the old file, now collapsed to a single line at the top of
`it('toggles quick preview...')`) is reduced to using the same
harness, so the new tests and the existing test share a single source
of truth for the agent-result fixture shape.

The four new tests:

1. `closes without applying or cleaning up when Escape is pressed`
   (`:44-64`): writes `\x1B`, asserts `closeArenaDialog` called once,
   and the load-bearing negatives `applyAgentResult` and
   `cleanupArenaRuntime` *not* called. This is the right shape for
   pinning Escape semantics — `not.toHaveBeenCalled()` blocks a future
   regression that would silently start applying on Escape.

2. `discards results without applying changes when x is pressed`
   (`:66-88`): writes `x`, asserts
   `cleanupArenaRuntime).toHaveBeenCalledWith(true)` (exact-arg match
   on the `true` flag), close called, apply *not* called. The exact-arg
   match on `true` is the load-bearing assertion — if the dialog
   started passing `false`, worktrees would silently leak.

3. `applies the highlighted successful agent when Enter is pressed`
   (`:90-114`): writes `\r`, asserts
   `applyAgentResult).toHaveBeenCalledWith('model-1')` and
   `cleanupArenaRuntime).toHaveBeenCalledWith(true)` — pins both the
   apply target by id and the post-apply cleanup contract.

4. `ignores Enter when the highlighted agent is not selectable`
   (`:116-138`): seeds the harness with a `FAILED` agent, writes
   `\r`, then awaits one tick and asserts *all three* mocks are not
   called. This is the most defensive arm — a regression here would
   apply a failed agent's (broken) diff to the user's worktree.

## Risks / nits

- All four tests use `await waitFor(...)` for the positive arms and a
  bare `await new Promise(resolve => setTimeout(resolve, 0))` for the
  negative arm in test #4. The latter is a single tick — fine for the
  current synchronous Ink update path, but a future refactor that adds
  any async delay between key event and decision could let a positive
  call leak through. A `setTimeout(50)` or a stronger `waitFor` with a
  short timeout would be more robust.
- `createAgentResult` defaults `status: AgentStatus.IDLE`; the FAILED
  arm explicitly overrides. No coverage for `RUNNING` or other
  intermediate states — would the Enter-on-RUNNING case be a hang or a
  no-op? Not covered, but probably out of scope.

## Verdict

**merge-as-is** — pure test addition, correctly factored shared
harness, four genuine behaviors locked in by exact-arg and
not-called assertions. Test-only change; no production-code risk.
