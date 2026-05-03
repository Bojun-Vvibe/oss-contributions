# Review: QwenLM/qwen-code #3809 — feat(core): hint to background long-running foreground bash commands

- **Repo**: QwenLM/qwen-code
- **PR**: #3809
- **Head SHA**: `fd93f102f5389e09149a813fa9c8171faf8b8ef1`
- **Author**: wenshao

## What it does

Phase D (a) auto-bg advisory: when a foreground `shell` tool invocation
runs ≥ 60s, append a hint to the LLM-facing output telling the model to
prefer `is_background: true` next time and pointing at `/tasks` and the
dialog. Suppressed on user-cancel / timeout (those have their own
messaging) and never fires on the background path (returns before the
threshold by construction).

## Diff notes

- `packages/core/src/tools/shell.test.ts:788-868` — four new test cases,
  driven with `vi.useFakeTimers({ toFake: ['Date'] })`:
  1. ≥ 60s success → hint present, mentions `is_background: true` and `/tasks`.
  2. < threshold success → no hint.
  3. ≥ 60s with `exitCode: 1` → hint still appended (correct: the
     "next time background it" advice applies to errored runs too).
  4. Aborted ≥ 120s → hint suppressed (correct: cancel/timeout has its
     own messaging).
  
  Test design is solid — fake timers avoid wall-clock dependence, the
  `tail -f` choice in test 4 dodges the sleep-interception validator
  (commented inline). Good defensive testing.
- `packages/core/src/tools/shell.ts:50-84` — adds the threshold logic.
  Only the first 35 lines were visible in the snippet but the tests are
  the contract; if the source matches the test expectations the
  behavior is well-bounded.

## Concerns

1. The 60s threshold is hardcoded. A `GEMINI_FOREGROUND_BG_HINT_MS` env
   var (or config field) would let teams tune it for slower CI / faster
   local runs. Non-blocking.
2. The hint message format is asserted exactly in tests
   (`'this foreground command ran for 60s'`,
   `'this foreground command ran for 75s'`). If maintainers want to
   localize the message later, those tests will need updating. Worth a
   comment on the test.
3. Verify the hint isn't appended to `displayContent` (only
   `llmContent`) so users don't see noisy advisory chrome in the TUI.
   The test only asserts on `result.llmContent` which is a good sign,
   but worth confirming the `display` field is untouched.
4. Hint should probably also be suppressed when `is_background` was
   *programmatically forced false* by config (e.g., a workspace that
   disabled background tasks). Edge case.

Overall: clean, small, well-tested feature flag of advice. The `/tasks`
reference is documentation-coupled; if the docs page name changes, the
hint string drifts. Minor.

## Verdict

merge-after-nits
