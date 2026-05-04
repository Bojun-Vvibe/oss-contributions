# openai/codex #20923 — Add plugin ID to skill analytics

- SHA: `124b5ab096c62dce8c1267a6ac4fea0a7f0a1185`
- State: OPEN, +190/-20 across 27 files

## Summary

Threads a `plugin_id: Option<String>` field through the skill-invocation analytics pipeline: protocol struct, fact ingestion, event params, reducer, and tests. When a skill is invoked via a plugin (path containing `/plugins/cache/<plugin_id>/...`), the resolved id flows into the emitted event payload; for non-plugin invocations the field is `null`.

## Notes

- `codex-rs/analytics/src/events.rs:80-86`: new field `plugin_id: Option<String>` on `SkillInvocationEventParams` is correctly `Option`-wrapped, so existing consumers ignoring it won't break.
- `codex-rs/analytics/src/facts.rs:173-...`: `SkillInvocation` struct gains `plugin_id`; this is a public type, so any external constructor sites in the same crate need updating — the 27-file fan-out suggests the author did this work consistently.
- `codex-rs/analytics/src/analytics_client_tests.rs:1819` adds `plugin_id: None,` to the existing reducer test, and a new test (`reducer_includes_plugin_id_for_plugin_skill_invocations`) asserts `payload[0]["event_params"]["plugin_id"] == "sample@test"`. The "user" scope in that test uses a path under `/Users/abc/.codex/plugins/cache/test/sample/skills/doc/SKILL.md` — make sure the plugin_id resolution helper is being invoked in the test path, not just hard-coded into the input. The test does set `plugin_id: Some("sample@test")` directly on the input, which means it tests serialization/forwarding, not derivation. A separate test should verify that derivation logic in the call site that *constructs* `SkillInvocation` from a path.
- 27 files for adding one optional field is on the high side; a bulk-edit gate-style classification (mechanical adds vs semantic) in the PR body would help reviewers.

## Verdict

`merge-after-nits` — add a unit test for the path-to-`plugin_id` derivation (or link to the call site that does it), and call out in the PR body which files are mechanical struct-init updates vs semantic logic changes.