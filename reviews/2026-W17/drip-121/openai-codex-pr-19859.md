# openai/codex #19859 — Show plugin hooks in plugin details

- URL: https://github.com/openai/codex/pull/19859
- Head SHA: `a7674d663750d82a612f59e30a36df0f15e31648`
- Diff: +298/-5 across 14 files (protocol schemas, app-server processor, core plugin manager, and tui details popup).

## Context / problem

Sibling PR #19705 added discovery for hooks bundled with plugins, but the `/plugins` detail view in the TUI/app-server still only renders skills, apps, and MCP servers. Users can't see which hooks a plugin provides without inspecting the plugin source on disk. This PR closes the visibility gap symmetrically across protocol → core → app-server → TUI.

## What the fix does

- New protocol type `PluginHookSummary { eventName: HookEventName, handlerCount: u32 }` in `app-server-protocol/src/protocol/v2.rs:+9` plus generated TS at `schema/typescript/v2/PluginHookSummary.ts`.
- `PluginDetail` gains a required `hooks: Vec<PluginHookSummary>` field — the JSON schemas (`codex_app_server_protocol.schemas.json`, `codex_app_server_protocol.v2.schemas.json`, `v2/PluginReadResponse.json`) all add `"hooks"` to the `required` array, plus a `HookEventName` enum: `["preToolUse", "permissionRequest", "postToolUse", "sessionStart", "userPromptSubmit", "stop"]` (drives the schema's enum constraint downstream).
- `core/src/plugins/manager.rs` (+44) adds the summarization helper that walks bundled plugin hooks, groups by `eventName`, counts handlers per event.
- `app-server/src/codex_message_processor/plugins.rs` (+10) wires the new field into the `plugin/read` response.
- `app-server/tests/suite/v2/plugin_read.rs` (+44) adds end-to-end coverage asserting the wire shape includes `hooks` with correct counts.
- `core/src/plugins/manager_tests.rs` (+44) covers the summarization helper directly.
- `config/src/hook_config.rs` (+11) exposes whatever the manager needs to enumerate handlers (the precise API change is small but lives in the public hook-config surface).
- TUI: a `Hooks` row in the `/plugins` detail popup renders the summary list.

## Specific references

- `app-server-protocol/src/protocol/v2.rs:+9` — the new struct. `eventName: HookEventName` + `handlerCount: u32` is the right minimal shape — tells you what fires and how many handlers without leaking handler bodies or paths.
- `schema/json/v2/PluginReadResponse.json:+11..+17` — `HookEventName` enum is hard-coded to six strings. If a future hook event is added to `HookEventName` (Rust enum), this generated schema must be regenerated; otherwise external consumers parsing strict-enum reject the new variant. Worth a one-line comment in `hook_config.rs` noting "regenerate JSON schemas after adding a HookEventName variant" near the enum declaration.
- `PluginDetail` `required` array gains `"hooks"` (`schemas.json:+11530, +11567`; `v2.schemas.json:+8204, +8241`; `PluginReadResponse.json:+72, +109`). This is a **breaking change for external strict-schema consumers** parsing `PluginReadResponse`: a client validating against a published v2 schema rev that doesn't have `hooks` in `required` will accept old responses but a client validating against the new schema rev will *reject* responses from older codex servers. There's no `BREAKING:` callout in the PR title or body. For an experimental v2 protocol this is probably acceptable, but should be flagged in release notes.
- `app-server/tests/suite/v2/plugin_read.rs:+44` — end-to-end assertion. Pinning the wire shape at the integration boundary (rather than just the manager unit test) is the right call because the JSON-schema/TS-binding generation pipeline is what most consumers actually depend on.

## Risks / nits

- Field is `required` rather than `optional` in the schema. For an experimental v2 protocol this is acceptable but should be called out as `BREAKING:` for downstream schema consumers who pin to a specific revision.
- The `handlerCount: u32` carries no information about *which* handler files implement the hook — fine for a summary view, but if users want to debug "why isn't my preToolUse hook firing", the natural next click is "show handler paths" which this PR doesn't enable. Out of scope for this PR but worth a follow-up.
- TUI row rendering: not visible in the diff slice I have, but a hook list with a dynamic event-name set could overflow the popup layout. Worth manually verifying with a plugin that registers all 6 event types.

## Verdict

`merge-after-nits` — symmetric end-to-end visibility fix with the right test coverage at both the unit (`manager_tests.rs`) and integration (`plugin_read.rs`) layers, but the `required: ["hooks"]` schema change should be called out as `BREAKING:` for external v2 consumers and the `HookEventName` enum should grow a "regenerate schemas after adding a variant" comment so the next contributor doesn't ship a Rust-only addition that rejects at the wire layer.
