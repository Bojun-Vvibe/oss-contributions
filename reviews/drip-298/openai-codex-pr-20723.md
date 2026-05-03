# Review: openai/codex PR #20723 — Clarify local and remote plugin selectors

- **Head SHA:** `f45d2ef31638a5ab3e7d95ddb0ef62147ee17852`
- **State:** CLOSED
- **Size:** +381 / -192, 17 files
- **Verdict:** `needs-discussion`

## Summary
Renames the `PluginInstallParams` / `PluginReadParams` selectors from
`marketplacePath` + `pluginName` (with optional `remoteMarketplaceName`) to
explicit `localMarketplacePath` + `localPluginName` + `remoteMarketplaceName`
+ `remotePluginId`. Touches the JSON schema, v2 schema, TypeScript types,
Rust protocol structs, app-server message processor, TUI callers, integration
tests, and the generated Python SDK. PR is currently CLOSED — review captures
the design intent for future revival.

## Strengths
- Schema rename is internally consistent across all four artifacts:
  `schema/json/ClientRequest.json:2122`,
  `schema/json/codex_app_server_protocol.schemas.json:12031`,
  `schema/json/codex_app_server_protocol.v2.schemas.json:8684`, and
  `schema/json/v2/PluginInstallParams.json:5`. No drift between v2 and v1
  flat schemas.
- The previously-required `pluginName` is now optional in both params types
  (the `required` block is removed). This correctly reflects the
  local-vs-remote disjunction: callers must supply *either* a local pair *or*
  a remote pair.
- Generated Python types at `sdk/python/src/codex_app_server/generated/v2_all.py`
  (+29/-7) update cleanly.
- TUI call sites at `tui/src/app/background_requests.rs:1-3`,
  `tui/src/app/event_dispatch.rs:1-3`, and `tui/src/chatwidget/plugins.rs:1-3`
  all touch only the field names — no semantic changes leak into the UI.

## Concerns

1. **No validation that exactly one selector pair is set.** The schema marks
   all four fields as optional, but the protocol semantics require either
   `(localMarketplacePath, localPluginName)` *or*
   `(remoteMarketplaceName, remotePluginId)`. Without a server-side check,
   callers can send all four (ambiguous), neither (no-op), or mismatched
   pairs. The Rust enum approach (`enum PluginSelector { Local{...},
   Remote{...} }`) would encode this in the type system. Worth doing before
   wire compatibility solidifies.

2. **Breaking schema change with no version bump signal.** The old
   `pluginName` field is silently gone. Any external client built against the
   pre-rename schema (including third-party SDKs not generated from this
   repo's schemas) will break with a deserialization error. The PR body
   doesn't mention a migration story or schema version bump.

3. **`README.md` only gets +2/-2** at `app-server/README.md`. If this is the
   public docs surface for the protocol, it should be expanded with the
   selector-pair semantics, not just the field rename.

4. **`plugin_install.rs` and `plugin_read.rs` integration tests** at
   `app-server/tests/suite/v2/` are net-positive (+89/-62) but I'd want to see
   a dedicated test for the "both pairs set" and "no pairs set" error paths
   before merge — the current tests look like rename-only.

5. **`common.rs:+3/-2`** is suspiciously small for a "rename a shared
   struct" change. Confirm there's no straggler caller still using the old
   `marketplace_path` field name in a non-schema location (search for
   `marketplace_path` and `plugin_name` snake_case across the workspace).

## Verdict rationale
The rename clarifies a confusing API surface and the cross-artifact
consistency is good. But the lack of mutual-exclusion enforcement in the
type/schema layer leaves the door open for the same kind of ambiguity the
rename is trying to fix. Worth reopening with a `PluginSelector` enum and
explicit `400` on ambiguous selectors.
