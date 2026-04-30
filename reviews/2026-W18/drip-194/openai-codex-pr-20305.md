# openai/codex PR #20305 — fix(exec-policy) use is_known_safe_command less

- PR: https://github.com/openai/codex/pull/20305
- Head SHA: `116859a1fc395fe826405a4d2463692200a74ef7`
- Files touched: `codex-rs/core/src/exec_policy.rs` (+8/-4), `codex-rs/core/src/exec_policy_tests.rs` (+67/-0), `codex-rs/core/tests/suite/approvals.rs` (+19/-0)

## Specific citations

- The load-bearing change is moving the `is_known_safe_command` short-circuit *below* the sandbox-protection computation at `exec_policy.rs:594-617`. Previously (deleted lines `:594-597`) any known-safe command (`echo`, `ls`, etc.) returned `Decision::Allow` unconditionally if not parsed via the complex path. New gate at `:606-612`:
  ```rust
  if is_known_safe_command(command)
      && !used_complex_parsing
      && (approval_policy == AskForApproval::UnlessTrusted
          || environment_lacks_sandbox_protections)
  ```
  i.e. the auto-allow now requires *either* the most permissive approval policy `UnlessTrusted`, *or* the environment genuinely has no sandbox (the Windows ReadOnly carve-out). Under `OnRequest` with a real workspace-write sandbox, even `echo hello` now flows through the prompt path.
- Test 1 at `exec_policy_tests.rs:1109-1124` (`known_safe_on_request_still_prompts_for_restricted_sandbox_escalation`) locks the new behaviour: `OnRequest` + workspace-write + `RequireEscalated` + `echo hello` ⇒ `Decision::Prompt`.
- Test 2 at `:1213-1233` extends the existing `RequireEscalated` matrix: under `OnRequest` the response is `NeedsApproval { reason: None, proposed_execpolicy_amendment: Some(["echo", "hello"]) }` — and crucially, with `Granular { sandbox_approval: false, ... }` at `:1236-1259` the response flips to `Forbidden { reason: REJECT_SANDBOX_APPROVAL_REASON }`.
- End-to-end approval scenario at `tests/suite/approvals.rs:967-984` adds `known_safe_escalation_on_request_requires_approval` with `model_override: Some("gpt-5.2")` and `Outcome::ExecApprovalWithAmendment { decision: ReviewDecision::Denied, ..., expected_execpolicy_amendment: Some(&["echo", "known-safe-escalation"]) }` — locking that the user-deny path produces `CommandFailure` with stderr containing `"rejected by user"`.

## Verdict: merge-as-is

## Concerns / nits

1. **The semantic shift is a real user-visible behaviour change**: under `OnRequest` (the default for many users), a previously-silent `echo hello` to a sandbox-write region will now prompt. This is the *correct* tightening — known-safe was being used as a sandbox bypass — but it deserves a CHANGELOG / release-notes entry naming the new prompting frequency so the user doesn't think the agent regressed. The PR title `fix(exec-policy) use is_known_safe_command less` is the right shape but more users will feel this than read the title.
2. The `Granular { sandbox_approval: false, ... }` + `RequireEscalated` test at `:1236-1259` produces `Forbidden` rather than `Prompt` — this is consistent with the existing matrix at the unmatched-command equivalent at `:1106` and is the right shape (granular config that explicitly disables sandbox-approval should reject not prompt).
3. The fix is correctly *additive* on test coverage (no existing test was deleted or weakened) — the two new exec-policy tests and the one suite scenario flag the new behaviour at three layers (decision unit, requirement assertion, end-to-end approval).
4. `used_complex_parsing` semantics preserved — still gates the auto-allow correctly so no regression on the parser-evasion class that #20277 hardened.
