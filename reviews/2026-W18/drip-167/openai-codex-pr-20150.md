# openai/codex PR #20150 — Add remote plugin skill read API

- Repo: openai/codex
- PR: https://github.com/openai/codex/pull/20150
- Head SHA: `3598e4e8acd6600fd0c9459d66bc3e7606bf21df`
- Author: xli-oai
- Size: +466 / −11, 17 files

## Context

Remote-marketplace plugins ship skill metadata that the desktop UI wants to
render *before* the user installs the plugin locally — there's no
on-disk `SKILL.md` to read, so the UI today has nothing to display in the
"skill details" modal. This PR adds a new app-server method
`plugin/skill/read` that takes `(remoteMarketplaceName, pluginName,
skillName)` and returns `{ contents: string | null }`, sourcing the markdown
from the plugin-service skill detail endpoint. The bulk of the diff is
generated schema fan-out (JSON + TypeScript + v2 variants) plus a wired
handler in `core-plugins`, an `app-server` dispatch, and an integration
test under `tests/suite/v2/plugin_read/`.

## What the diff actually adds

- Two new schema files at
  `app-server-protocol/schema/json/v2/PluginSkillReadParams.json` and
  `PluginSkillReadResponse.json` defining the request shape
  `{ remoteMarketplaceName, pluginName, skillName }` (all required) and
  the response `{ contents: string | null }`.
- The same shapes get added to the consolidated bundles
  `codex_app_server_protocol.schemas.json` and
  `codex_app_server_protocol.v2.schemas.json` (plus their `ClientRequest`
  enumerations gain `"plugin/skill/read"` entries).
- The TypeScript surface gains
  `schema/typescript/v2/PluginSkillReadParams.ts` /
  `PluginSkillReadResponse.ts` and the `ClientRequest` union type at
  `schema/typescript/ClientRequest.ts` is regenerated to include
  `{ "method": "plugin/skill/read", id: RequestId, params:
  PluginSkillReadParams }`. (PR body cites
  `just write-app-server-schema` as the regen command.)
- The integration test
  `tests/suite/v2/plugin_read/plugin_skill_read_reads_remote_skill_contents_when_remote_plugin_enabled`
  pins end-to-end behavior. The hyphenated test name implies the
  contract is "if the remote plugin is enabled, the call returns the
  remote skill markdown" — i.e. the read is gated on the remote plugin
  being enabled in the local config.

## What I want to see before merge

1. **Response shape is `contents: string | null` but the meaning of `null`
   is unclear.** Three reasonable interpretations exist: (a) the skill
   exists but has empty markdown, (b) the skill doesn't exist on the
   remote, (c) the plugin-service call failed. Each should map to a
   different UI state (render empty, render "skill not found", render
   "fetch failed"). Either widen the response to a discriminated union
   (`{ contents: string } | { error: "not_found" | "fetch_failed" |
   "disabled" }`) or document the single null-meaning in a JSON-schema
   `description` so clients don't paper over the ambiguity with a generic
   "no content available".

2. **No required-field validation visible in the schema for the
   plugin-service auth path.** A read against an arbitrary
   `(remoteMarketplaceName, pluginName, skillName)` triple presumably
   needs the user to have authenticated with that marketplace. If the
   handler silently returns `null` when auth is missing, a UI that just
   shows "no content" will mask a config error. The integration test name
   includes `_when_remote_plugin_enabled` which suggests the gate exists,
   but there should also be a negative test
   (`_returns_disabled_error_when_remote_plugin_disabled` or similar)
   pinning the no-auth / disabled path.

3. **Caching strategy isn't called out.** Skill markdown is conceptually
   immutable per plugin version. If the desktop UI opens the modal
   repeatedly (browse → close → reopen), this round-trips to the
   plugin-service every time. Even a 60-second in-process cache keyed on
   `(marketplace, plugin, skill, version)` would cut the latency for
   the obvious use case. Worth a TODO at minimum.

4. **Schema params and TS params have field-order divergence.** The
   generated TS at `schema/typescript/v2/PluginSkillReadParams.ts` reads
   `{ remoteMarketplaceName: string, pluginName: string, skillName:
   string }` while the JSON schema lists them as `pluginName`,
   `remoteMarketplaceName`, `skillName`. JSON properties are unordered so
   this is harmless on the wire, but the diff diverges from how the
   adjacent generated files read top-to-bottom; if the regenerator is
   sensitive to source declaration order in the Rust struct, future
   regens may flip the order again and produce a noisy diff. Worth
   pinning the struct field order to match JSON-Schema-required order
   for reviewer-friendliness.

5. **No error-class entry for `plugin/skill/read` failures.** The PR
   doesn't show the `app-server` dispatcher diff or the error variants —
   I'd want to see what happens when the plugin-service returns 5xx, when
   the marketplace is unknown, and when the skill name doesn't exist on
   the remote. Each should be a distinct error class so the UI can
   differentiate "transient retry" from "permanent missing".

## Suggestions

- Replace `contents: string | null` with a discriminated union covering
  the not-found / disabled / fetch-failed cases, or document the single
  null meaning explicitly in the JSON-schema `description`.
- Add a `_when_remote_plugin_disabled` negative integration test.
- Add a short-TTL in-process cache keyed by `(marketplace, plugin, skill,
  version)` for the modal-reopen pattern.
- Ensure the Rust struct field order matches the JSON-required order so
  regen doesn't introduce noisy diffs.
- Add an `ErrorCode::PluginSkillReadFailed` (or whatever the project's
  convention is) and exercise it in a test.

## Verdict

**merge-after-nits** — the protocol surface is sensibly small, the
generated schema fan-out looks complete, and the integration test pins
the happy path. The nits are around making the failure modes legible
to the UI rather than collapsing them to `null`.
