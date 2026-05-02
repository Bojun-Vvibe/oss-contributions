# openai/codex #20756 — [codex] support PreToolUse allow and ask decisions

- URL: https://github.com/openai/codex/pull/20756
- Head SHA: `ccc920a06254bde1066640d6584ef1c2cf3235ca`
- Author: @abhinav-oai
- Stats: substantial — touches `hook_runtime.rs`, `mcp_tool_call.rs`,
  and tests

## Summary

Extends the PreToolUse hook contract: in addition to `block`, hooks may
now return `Allow { reason }` or `Ask { reason }`. `Allow` short-circuits
the MCP approval pipeline (no policy auto-approve check, no remembered
session approval, no guardian, no user prompt). `Ask` does the
opposite — forces a user prompt even if policy/cache/guardian would have
auto-approved.

## Specific feedback

- `codex-rs/core/src/hook_runtime.rs:140-186` — return type changes from
  `Option<String>` (= block reason) to
  `Result<Option<PreToolUsePermissionDecision>, String>` (Err = block,
  Ok(Some) = explicit decision, Ok(None) = no opinion). Cleanly encodes
  the three states; callers will need to update — confirm every
  `run_pre_tool_use_hooks` call-site has been migrated (search the
  branch for the old signature).
- `codex-rs/core/src/mcp_tool_call.rs:918-934` — `maybe_request_mcp_tool_approval`
  now `#[cfg(test)]` and just delegates to the `_with_pre_tool_use_decision`
  variant with `None`. Reasonable shim, but it's load-bearing for tests
  only — consider just inlining the `None` at every test call-site to
  avoid two functions that drift.
- `codex-rs/core/src/mcp_tool_call.rs:949-957` — `Allow` short-circuit
  returns `None` (= no decision needed → tool runs). Correct, but the
  bypass is total: it skips `requires_mcp_tool_approval(annotations)`,
  guardian routing, AND any persistent/session approval bookkeeping.
  That's the documented intent for `Allow`, but worth a doc comment
  inline so a future reader doesn't tighten the gate.
- `codex-rs/core/src/mcp_tool_call.rs:986-1030` — `force_user_prompt`
  is threaded through every short-circuit (`auto_approved_by_policy`,
  remembered approval, permission_request_hooks, guardian). The negation
  pattern (`&& !force_user_prompt`) is consistent and easy to audit. One
  nit: the early-return at line 989 (`approval_required && approval_mode
  != Prompt && !force_user_prompt`) is now a triple-conjunction — pull
  it into a named bool (`let can_skip_prompt = ...`) for readability.
- `codex-rs/core/src/mcp_tool_call.rs:996-998` — `monitor_reason` now
  seeds from `pre_tool_use_permission_decision.reason()`. That's the
  right place to surface the hook's rationale to the auto-approve
  monitor; no concerns.
- `codex-rs/core/src/mcp_tool_call_tests.rs:2020-2061` — new test
  `pre_tool_use_allow_skips_mcp_approval_flow` is precisely the
  load-bearing case (Prompt mode + dangerous-tool annotations + Allow →
  no prompt). Missing a symmetrical `Ask` test that proves a permissive
  policy is overridden — please add before merge.

## Verdict

`merge-after-nits` — coherent extension to a tightly-coupled approval
state machine. The missing `Ask` regression test and the readability
nit on the triple-conjunction are the only real asks. Security model is
sound: `Allow` is documented bypass, `Ask` is documented escalation.
