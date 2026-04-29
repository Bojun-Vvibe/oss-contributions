# QwenLM/qwen-code #3723 — feat(core): add shared permission flow for tool execution unification

- **PR:** https://github.com/QwenLM/qwen-code/pull/3723
- **Head SHA:** `451fb0cb6b10648400948ee7186e11c937d7a259`
- **Size:** +395 / −0 (4 files: 1 doc + 1 source + 1 test + 1 export)
- **Tracks:** Issue #3247 ("Tool Execution Unification")

## Summary
First step in a refactor sequence: introduces a new `permissionFlow.ts` module exporting four pure functions (`evaluatePermissionFlow`, `needsConfirmation`, `isPlanModeBlocked`, `isAutoEditApproved`) that consolidate scattered permission-decision logic for tool execution into one testable surface. Adds `permissionFlow.test.ts` with 17 test cases covering the matrix. Exports the module from `packages/core/src/index.ts`. Adds a one-line `TOOL_EXECUTION_UNIFICATION.md` tracking doc.

The "unification" payoff lands later — this PR is the **shared primitive** that Interactive / Non-Interactive / ACP modes will all eventually call.

## Specific observations

1. **`evaluatePermissionFlow` (the load-bearing function)** at `permissionFlow.ts` consumes `(config, invocation, toolName, params)` and returns `{ finalPermission, denyMessage?, pmCtx, pmForcedAsk? }`. The decision tree per the test file:
   - `defaultPermission === 'deny'` → returns `{deny, "tool's default permission is 'deny'"}` (test `permissionFlow.test.ts:39-49`)
   - PM has relevant rules → returns PM verdict; if PM denies, includes `"denied by permission rules"` + `"Matching deny rule"` in denyMessage (test `:53-79`)
   - PM has no relevant rules → returns `{ask}` (test `:81-101`)
   - PM has matching ask rule → returns `{ask, pmForcedAsk: true}` (test `:103-127`)
   
   The function is correctly modelled as pure (mocked PM with `vi.fn()`, no side effects). Good architectural shape for the unification goal.

2. **`needsConfirmation` at `:131-150`** — the test pins behavior:
   - YOLO mode + non-`ask_user_question` tool → `false`
   - YOLO mode + `ToolNames.ASK_USER_QUESTION` → `true` (special case, ask_user_question always confirms)
   - Non-YOLO + `'ask'` or `'default'` permission → `true`
   - Non-YOLO + `'allow'` or `'deny'` → `false`
   
   The ASK_USER_QUESTION special-case in YOLO is the right call (YOLO bypasses normal approval but the user explicitly invoked a question tool, so the tool's purpose is asking). However the test doesn't pin the case `(YOLO, 'allow', some_tool)` — does YOLO override `'allow'` to also-no-confirm? Probably yes (allow already says no-confirm), but the matrix gap is worth filling.

3. **`isPlanModeBlocked` at `:152-...`** — tests pin:
   - Plan mode + non-info tool → blocked
   - Plan mode + info-type tool → not blocked
   - Plan mode + exit_plan_mode → not blocked (special case)
   - Plan mode + ask_user_question → not blocked (special case)
   - Not plan mode → never blocked
   
   The two special cases (exit_plan_mode, ask_user_question) are encoded as positional booleans (`isPlanModeBlocked(planMode, isExitPlanMode, isAskUserQuestion, confirmationDetails)`) — this signature shape will become unwieldy if a third "plan-mode-allowed" tool is added (e.g. read_file). A bag-of-flags object or a `PlanModeAllowedTool[]` constant would scale better. Minor.

4. **`isAutoEditApproved` at `:...`** — tests pin AUTO_EDIT mode auto-approves `'edit'` and `'info'` confirmation types but NOT `'exec'`. That asymmetry is correct (auto-edit shouldn't run shell commands) and is exactly the bug class that ungated AUTO_EDIT modes have shipped in other agents. Good defensive default.

5. **`TOOL_EXECUTION_UNIFICATION.md` is a one-line file at the repo root.** Repo-root tracking docs for in-flight refactors are a smell — they accumulate, never get deleted, and clutter the project root. Either move under `docs/refactor/` or delete and let the issue tracker carry the context (Issue #3247 is already linked). The file contains literal `\n\n` escape sequences instead of actual newlines, suggesting it was written via a single-line shell heredoc — minor but worth fixing before merge.

6. **No callers wired up yet.** The four functions are exported from `packages/core/src/index.ts` but no existing tool-execution path (Interactive, Non-Interactive, ACP) currently calls them. That's the right shape for an incremental refactor (introduce the primitive, prove it via tests, then migrate callers in follow-up PRs), but the PR title "tool execution unification" overstates what landed — this is the *foundation* for unification, not unification itself. PR title could be `feat(core): add shared permissionFlow primitive for upcoming tool-execution unification` to set expectations.

## Verdict: `merge-after-nits`

Excellent test surface (17 cases pinning the decision matrix), pure functions, correct asymmetric defaults (AUTO_EDIT excludes exec). The marker file is sloppy and the PR title oversells; both are easy fixes.

## Recommended actions
- **(Required)** Fix `TOOL_EXECUTION_UNIFICATION.md` — it contains literal `\n\n` strings instead of real newlines. Either fix the file content or move it under `docs/refactor/` or delete it and rely on Issue #3247.
- **(Recommended)** Refine PR title to reflect that this lands the primitive, not the unified callers — sets reviewer expectations for the follow-up PRs.
- **(Recommended)** Migrate `isPlanModeBlocked`'s positional-boolean signature to a `{planMode, exitPlanMode, askUserQuestion, confirmationDetails}` object before more tools need plan-mode exemptions — easier to extend.
- **(Optional)** Add a YOLO-mode test cell for `(YOLO, 'allow', some_tool)` to lock that no-confirm dominates over the ASK_USER_QUESTION exception only when the tool name actually matches.
