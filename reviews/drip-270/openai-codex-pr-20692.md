# openai/codex #20692 — Support PreToolUse additionalContext

- URL: https://github.com/openai/codex/pull/20692
- Head SHA: `f9dfd3d50c3bdffbad5a1f16aaed3983fe28ce82`
- Author: @abhinav-oai
- Stats: +261 / -25 across 5 files

## Summary

Promotes the previously-rejected `additionalContext` field on the
PreToolUse hook output from "unsupported" to a first-class signal:
hook scripts can now return `{"hookSpecificOutput": {"additionalContext":
"..."}}` (allowed alone or alongside `permissionDecision: "deny"`),
and the runtime injects that context into the next model request via
`record_additional_contexts`. Two integration tests cover both the
allow-with-context and deny-with-context shapes.

## Specific feedback

- `codex-rs/hooks/src/engine/output_parser.rs:96-100` — the parser now
  threads `additional_context` out of `hook_specific_output` cleanly.
  Removing `output.additional_context.is_some()` from
  `use_hook_specific_decision` (line 102, deleted) is the key change:
  having only `additionalContext` is no longer treated as "unsupported".
  Confirm there is a corresponding deletion in
  `unsupported_pre_tool_use_hook_specific_output` (around line 339-345
  per the diff truncation) — if that branch still rejects pure
  `additionalContext` payloads, the parser will succeed but the validator
  will then reject them.
- `codex-rs/hooks/src/events/pre_tool_use.rs:*` — +87/-16; the run-result
  shape now carries `additional_contexts: Vec<...>`. Make sure the
  field is added to any `Default` impls and to serde-skipped serialization
  paths used by the JSON-RPC event surface, otherwise older clients that
  decode the structure strictly may fail.
- `codex-rs/core/src/hook_runtime.rs:161-167` — `record_additional_contexts`
  is invoked **before** the `should_block` check. That is correct
  (blocked tools should still surface their reason as developer-visible
  context) and is exactly what the new
  `blocked_pre_tool_use_records_additional_context_for_shell_command`
  test asserts at `hooks.rs:1955-1990`.
- `codex-rs/core/tests/suite/hooks.rs:1851-1916` — the
  `pre_tool_use_records_additional_context_for_shell_command` test
  asserts that the second mock-server request contains the context
  string in the developer-role message_input_texts. That's the right
  invariant. Nit: the marker-file negative assertion only fires in the
  blocked variant — fine.
- `codex-rs/core/tests/suite/hooks.rs:1955-2003` — uses
  `PermissionProfile::Disabled` to ensure the policy itself doesn't
  block the command (so the only blocker is the hook). Good isolation.
- `codex-rs/core/src/tools/registry.rs:458-461` — only a stylistic blank
  line was removed inside `additional_contexts.clone()`; no functional
  change here, just churn. Keep or revert per project style.
- One thing to verify outside this diff: `additionalContext` on a
  PreToolUse hook is documented in the user-facing hooks reference. The
  PR does not appear to update the docs site.

## Verdict

`merge-after-nits` — feature is small, well-tested, and correctly
ordered (record-then-block). Verify the
`unsupported_pre_tool_use_hook_specific_output` branch was updated to
match the parser's relaxation, and ship a docs update for the hook
reference.
