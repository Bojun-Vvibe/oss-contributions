# openai/codex #20379 — Send external import completion for sync imports

- **Author:** alexsong-oai
- **SHA:** `9219e75`
- **State:** OPEN
- **Size:** +45 / -9 across 3 files (`app-server/README.md`,
  `app-server/src/message_processor.rs`,
  `app-server/tests/suite/v2/external_agent_config.rs`)
- **Verdict:** `merge-as-is`

## Summary

Closes a notification-shape gap in `externalAgentConfig/import` where the
server only emitted the `externalAgentConfig/import/completed` lifecycle
notification when the request payload contained at least one `Plugins` or
`Sessions` migration item. Synchronous-only imports (a single
`{itemType: "CONFIG"}` item, for example) returned the response and then
silently never emitted the completion event, leaving callers waiting on an
unresolvable promise. Fix at `message_processor.rs:1215,1233-1239` replaces
the two-arm `has_plugin_imports || has_session_imports` predicate with a
single `has_migration_items = !params.migration_items.is_empty()` early-exit
gate, then refactors the synchronous-completion check to a named
`has_background_imports` boolean so the invariant ("emit completion when
items present and no background work was queued") reads cleanly.

## Reasoning

Three things this PR gets right:

1. **The new gate is the correct invariant.** "Emit completion if any
   migration items were processed" matches both the README contract update
   at `app-server/README.md:225` and the existing async-completion path
   (which already fired for any item type that took the background route).
   The old code had two separate "is this kind worth notifying about?"
   predicates that were silently incompatible with the README's "after the
   full import finishes" promise.

2. **Locked behaviorally with a precise regression.** The new test
   `external_agent_config_import_sends_completion_notification_for_sync_only_import`
   at `tests/suite/v2/external_agent_config.rs:35-71` sends exactly the
   minimum-reproducer payload (`{"itemType": "CONFIG", "description": "Import
   config", "cwd": null}`), reads the response with `to_response` to confirm
   the typed shape, then uses
   `mcp.read_stream_until_notification_message("externalAgentConfig/import/completed")`
   with the standard `DEFAULT_TIMEOUT = 60s` envelope. The assertion is
   exact-method-match on the notification, so a future regression that
   accidentally changes the notification name will fail loudly rather than
   just timing out.

3. **README phrasing was updated in lockstep.** The doc change
   ("plugin or session imports" → "migration items") at
   `README.md:225` is small but it's the contract callers will read first;
   keeping it synchronized with the code matters more than the size of the
   diff suggests.

The dead `has_session_imports` local is correctly removed (it was only a
component of the deleted predicate); `has_plugin_imports` is kept because
it's still consumed downstream by the runtime-refresh path
(`needs_runtime_refresh` at `:1214`). The only question worth a sanity check
is whether the existing
`external_agent_config_import_sends_completion_notification_for_local_plugins`
test still passes unchanged — it should, because plugin imports
unconditionally took the notification path before, and the new gate
(`has_migration_items`) still fires for them. The diff doesn't touch that
test, so the assertion stands or falls on CI; reviewer should confirm green
before merging.

Net: tight 3-file fix with a regression test that exercises the failing
path. Ship it.
