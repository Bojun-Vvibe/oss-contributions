---
pr-url: https://github.com/openai/codex/pull/20471
sha: b52083146c2b
verdict: merge-as-is
---

# Stop emitting item/fileChange/outputDelta notifications

Server-side dead-code removal: deletes the `is_file_change_output: bool` parameter from `item_event_to_server_notification` at `app-server-protocol/src/protocol/event_mapping.rs:36`, drops the `FileChangeOutputDeltaNotification` import at `:13`, and adds `Deprecated legacy notification for apply_patch textual output. The server no longer emits this notification` doc-comments at six locations across the JSON/v2/typescript schema mirrors and the `server_notification_definitions!` macro arm at `protocol/common.rs:1400`. The notification *type* stays in the schema (so existing v0/v1 clients that decode the variant don't fail-on-unknown), but the server-side emit site is gone — the path through `item_event_to_server_notification` no longer dispatches to a `FileChangeOutputDelta` arm because the `is_file_change_output` discriminator that used to gate it is removed.

The retire-but-keep-the-type shape is correct: dropping the variant entirely would force every downstream client (TUI, IDE bridge, third-party `codex` MCP wrappers) to re-vendor their generated bindings on this release, and the variant-decode is essentially zero-cost on the wire because the server simply never emits the message. The schema doc-comment lands the contract in the four places downstream clients actually look (`schema/json/ServerNotification.json`, `codex_app_server_protocol.schemas.json`, `v2.schemas.json`, `schema/typescript/v2/FileChangeOutputDeltaNotification.ts`) so the deprecation is visible at every codegen surface, not just one.

The reason this is a `merge-as-is` rather than `merge-after-nits` despite the 489-line diff: every non-doc-comment change is a deletion, the deletion is type-checked (the `is_file_change_output` parameter going away forces every call site to drop the argument or fail to compile), the schema `description` field is the canonical surface for "this is deprecated, here's why" in the JSON-schema dialect being used, and the load-bearing `EventMsg::PatchApplyDelta` events are still flowing through the parent stream so consumers that wanted the delta information get it from the higher-fidelity event. The risk is low because the `outputDelta` was a textual-string view of structural data the consumer was already getting elsewhere — clients that rendered both were double-rendering, clients that rendered only this one were rendering a strictly-worse view of `apply_patch` output.

## what I learned
The right deprecation surface for a wire-protocol notification is "stop emitting at the server, keep the type-decode at every client, document at every codegen surface" — and the *failure mode* of skipping any one of those three (server still emits → wasted bandwidth + double-render; client drops decode → fail-on-unknown errors for users on stale clients; doc-comment only in the macro → SDK consumers reading the `.ts`/`.json` schemas don't see the deprecation) is exactly the kind of thing that gets discovered six releases later when someone files "why is my IDE plugin showing duplicate diff text."
