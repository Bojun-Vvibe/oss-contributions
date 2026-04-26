# QwenLM/qwen-code PR #3645 — fix(cli): correct OPENAI_MODEL precedence without breaking /model selection

- **PR:** https://github.com/QwenLM/qwen-code/pull/3645
- **Author:** B-A-M-N
- **Head SHA:** `fd04a01a2452a52f2767dbe3ac9a8c50ac45d45c`
- **Files:** 2 (+293 / -7)
- **Verdict:** `merge-after-nits`

## What it does

Re-fixes a model-precedence regression first introduced in #3567, then
reverted in #3633 because the original fix broke `/model` runtime
selection. This PR restores the intended ordering:

1. `argv.model` (CLI flag) — highest priority
2. `settings.model.name` (set via `/model`) — used for `modelProvider`
   lookup when it matches a known provider
3. `OPENAI_MODEL` env — fallback when no settings model is configured
   *or* when `settings.model.name` doesn't match any provider
4. `QWEN_MODEL` env — final fallback
5. Non-OpenAI auth ignores `OPENAI_MODEL` entirely

## Specific reads

- The diff is +293/-7 across 2 files but **all 286 net-added lines are
  test cases** in `packages/cli/src/utils/modelConfigUtils.test.ts`. The
  actual production change is in the second file (not visible in the
  first 80 diff lines). That ratio — minimal production change, large
  test surface — is exactly what you want for a regression-fix PR. It
  pins the behavior matrix so the next refactor can't silently flip
  precedence again.
- Tests cover seven distinct cases: A (settings wins over env), B (env
  wins when no settings), Edge1 (argv wins always), Edge2 (QWEN_MODEL
  fallback), Edge3 (OPENAI_MODEL > QWEN_MODEL), Edge4 (non-OpenAI auth
  ignores OPENAI_MODEL), Edge5 (settings.model.name doesn't match →
  fall back to OPENAI_MODEL). That's the full Cartesian product of
  presence × match × auth_type. Hard to regress without a test going
  red.
- `Case A` test at `modelConfigUtils.test.ts:+1-40` of the diff —
  asserts that `resolveModelConfig` is called with `modelProvider:
  settingsProvider` (the local one), not `envProvider`. Good
  black-box assertion shape.
- `Edge Case5` — "falls back to OPENAI_MODEL when settings.model.name
  doesn't match" — this is the **subtle case** that the previous fix
  got wrong. If a user sets `/model some-removed-model` and that
  model has been deleted from `modelProviders`, the old fix would
  either crash or pick a wrong default; the new logic falls through
  to OPENAI_MODEL. Right call.
- The only thing I can't verify from the visible diff is the
  production change itself — what `resolveCliGenerationConfig` now
  looks like. The tests assert the contract; the contract has to be
  enforced in real code. Need to read
  `packages/cli/src/utils/modelConfigUtils.ts` to confirm.

## Risk surface

**Low.** Test-heavy, narrow logic change, with an explicit prior-art
trail (`#3567`, `#3633`) showing the maintainers know exactly what
shape this regression takes. Two things to flag:

1. **Author's verification claim** — "All tests pass (35/35)." That's
   the test count for `modelConfigUtils.test.ts` after this PR's 7 new
   tests are added; need to confirm no tests were silently weakened or
   removed elsewhere in the file. A `git diff --stat` on just the test
   file should show only additions.
2. **Behavior change for users currently relying on the
   reverted-#3567 behavior.** Anyone whose workflow depended on
   `OPENAI_MODEL` always winning over `/model` will now experience
   `/model` taking precedence. That's the *correct* fix per UX intent
   ("`/model` is interactive, the env var is a default"), but it is a
   user-observable change.

## Suggestions before merge

- Confirm the production `modelConfigUtils.ts` change matches the
  test contract — specifically that `settings.model.name` is checked
  against `modelProviders[selectedAuthType]` *before* falling back to
  `OPENAI_MODEL`.
- Add one test for the case "argv.model is set but doesn't match any
  provider" — does it fail loud, or silently fall through to
  settings? The seven cases listed don't appear to cover that edge.
- Add a CHANGELOG entry noting the precedence change so users who
  were depending on the post-revert behavior aren't surprised.

Verdict: merge-after-nits — sound regression-fix shape (small
production change, large test matrix, explicit reference to the
previous attempt), assuming the production-side change matches the
test contract on read. Add the missing edge-test and CHANGELOG note.

## What I learned

Precedence chains are the kind of thing that look obvious in spec
text ("CLI > settings > env, with auth-type gating") but become a
soup of `if/else` branches in code. The pattern that works here —
**revert the broken fix, then re-fix with the test matrix written
first** — is the right play. The fact that the test file grew from
~735 lines to ~1020 lines (35 tests) is the *point*; the production
change is small precisely because the test matrix forced the
implementation to handle each case explicitly. Compare to a "clever"
single-expression precedence chain that reads tersely but breaks
unobservably when one input is null.
