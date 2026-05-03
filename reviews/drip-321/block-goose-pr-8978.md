# block/goose PR #8978 — fix schedule removal to keep recipes

- Link: https://github.com/block/goose/pull/8978
- SHA: `a94adcdae5a2a10811154f65af89315755b8efc3`
- Author: angiejones
- Stats: +60 / −14, 8 files

## Summary

Reverses an unsafe default: `scheduler.remove_scheduled_job(&id, true)` (true = "also delete the underlying recipe file") is changed to `false` in all three call-sites that handle user-initiated unschedule actions (CLI command, server REST route, agent tool). Renames the success messages from "deleted" to "removed" to reflect that only the schedule binding is gone. Adds a `test_remove_scheduled_job_respects_recipe_removal_flag` regression test that asserts both branches of the flag.

## Specific references

- `crates/goose-cli/src/commands/schedule.rs` L183: `scheduler.remove_scheduled_job(&schedule_id, true)` → `false`. CLI users running `goose schedule remove <id>` no longer lose their recipe file. Println at L186 changes to "Scheduled job '<id>' removed. Associated recipe was kept." — accurate and discoverable.
- `crates/goose-server/src/routes/schedule.rs` L208: same flip on the `DELETE /schedules/{id}` route. The OpenAPI doc text at L195 and `ui/desktop/openapi.json` L2752 are also updated from "deleted" to "removed", and `ui/desktop/src/api/types.gen.ts` L3770 mirrors the docstring change. Symmetrically updated — UI clients reading the response description will see the new wording, and the regenerated `types.gen.ts` confirms the OpenAPI/TS pair stayed in sync.
- `crates/goose/src/agents/schedule_tool.rs` L284: agent-tool path also flips to `false`. Success and error strings updated to "removed schedule"/"Failed to remove schedule". Important — the previous agent-facing text said "Successfully deleted job" which would have led the model to believe the recipe was gone too. The new wording is consistent with the new behaviour.
- `crates/goose/src/scheduler.rs` L1148–L1191: new `test_remove_scheduled_job_respects_recipe_removal_flag` covers both directions:
  - `remove_scheduled_job(..., false)` then `assert!(recipe_path.exists())`.
  - re-add, `remove_scheduled_job(..., true)`, then `assert!(!recipe_path.exists())`.
  This locks in the contract that the underlying API still supports both modes and only the call-sites changed defaults. Good test design.
- `ui/desktop/src/components/schedule/SchedulesView.tsx`: not in the head of the diff; presumably copy-only update to confirmation dialogs. Worth a quick eyeball that the user-facing "Remove schedule?" confirmation text matches the new semantics ("recipe file will be kept") so users don't expect the old destructive behaviour.
- `ui/desktop/src/i18n/messages/en.json`: ditto — translation keys for the confirmation copy should reflect the new behaviour. Reviewers: confirm both files actually updated the user-facing strings, not just symbol renames.

## Verdict

verdict: merge-after-nits

## Reasoning

This is the right fix for a footgun that made unscheduling unrecoverable. The API still supports both modes (verified by the new test), call-sites flip to the safer default, and the agent-facing text is corrected so the model stops claiming the recipe was deleted. Two nits: (1) confirm `SchedulesView.tsx` and the `i18n` JSON updates also reframe the confirmation dialog so users don't get a "delete recipe?" prompt that doesn't match reality; (2) consider a CLI flag (`--purge-recipe`) to expose the destructive path explicitly, since the feature still exists at the API layer but is no longer reachable from any user-facing surface. Not a blocker.
