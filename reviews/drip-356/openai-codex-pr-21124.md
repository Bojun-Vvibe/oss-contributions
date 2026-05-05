# openai/codex#21124 — feat: Add plugin share access controls

- Head SHA: `2558aafa2a1903fbbf0a8c92706ae83affd8e0c8`
- Author: xl-openai
- Verdict: **merge-after-nits**

## Summary

Extends `plugin/share/save` to take optional `discoverability`
(`LISTED`/`UNLISTED`/`PRIVATE`) and `shareTargets` (a list of
`{principalType, principalId}` tuples for `user`/`group`/`workspace`).
Adds a sibling `plugin/share/updateTargets` request so a plugin owner
can rotate ACLs without re-uploading the plugin payload.

## Specific references

- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:2200-2210`
  introduces the `PluginShareDiscoverability` enum with three string
  variants. Wire format is human-readable strings; that's fine but
  locks the API — adding a fourth state later is a versioning event.
- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:2218-2226`
  introduces `PluginSharePrincipalType` with `user`/`group`/`workspace`
  (note: lowercase, unlike the discoverability enum). Inconsistent
  casing across two enums introduced in the same PR.
- `codex-rs/app-server-protocol/schema/json/ClientRequest.json:2240-2261`
  adds `discoverability` and `shareTargets` to `PluginShareSaveParams`.
  Both are nullable; existing clients that omit them get the previous
  default behavior. Backward compatible.
- New `PluginShareUpdateTargetsParams` request body — keyed on
  `remotePluginId`, takes a fresh `shareTargets` list. PUT-style
  semantics (full replacement) — not documented in the schema.

## Reasoning

The protocol additions are clean and additive. The
`discoverability`/`shareTargets` decomposition matches conventional
ACL designs (visibility flag separate from explicit grant list) and
keeps the data model open to grant-with-role extensions later.

Nits worth fixing before merge:

1. **Enum casing inconsistency.** `PluginShareDiscoverability` is
   `LISTED`/`UNLISTED`/`PRIVATE` (SCREAMING) while
   `PluginSharePrincipalType` is `user`/`group`/`workspace` (snake).
   Pick one convention or document the rationale; future readers will
   re-litigate this every time they touch the file.

2. **`PluginShareUpdateTargetsParams` semantics undocumented.** Is the
   `shareTargets` array a full replacement, an additive merge, or a
   diff? The handler's behavior matters for clients building
   optimistic-UI ACL editors. Add a one-line description in the JSON
   schema (`description` field on the property).

3. **No `revoke` / removal primitive shown in the diff.** If
   `updateTargets` is full-replacement, then "remove user X from
   sharing list" is `updateTargets(currentList - {X})`, which forces
   the client to read-then-write and races other concurrent edits.
   Worth confirming the design intent.

The protocol is the right shape for what it's solving; the missing
descriptions and the casing nit are cheap to address.
