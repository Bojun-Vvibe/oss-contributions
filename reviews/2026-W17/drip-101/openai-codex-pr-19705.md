# openai/codex PR #19705 — Discover and manage hooks from plugins

- URL: https://github.com/openai/codex/pull/19705
- Head SHA: `d5539f437651ccecfff224ec4c88b9073d1a9fe9`
- Diff: +2565 / -236 across 64 files
- Author: openai member (abhinav-oai)

## Context / Problem

Hooks recently became a first-class extension surface (alongside skills),
but discovery was limited to the local config layer chain
(project / user / mdm / session-flags / legacy-managed-config). With
plugin support landing, plugins also want to bundle hooks — and app
clients need the same management UX as skills: list discovered hooks
including ones disabled by user config, plus a write API to flip the
disabled bit per-stable-key.

## Design

Three protocol additions visible in the schema diffs at
`codex-rs/app-server-protocol/schema/json/v2/`:

- `HooksListParams { cwds?: string[] }` — empty means current session
  cwd. Returning per-cwd is the right shape because hook-config layers
  vary by project location.
- `HooksListResponse` exposes discovered entries *including* disabled
  ones (PR body: "Filters disabled hooks out of runtime
  preview/execution while preserving them in list output"). This is
  the same shape skills already have, which is the contract you want.
- `HooksConfigWriteParams { key: string, enabled: bool }` — keyed by
  "stable hook keys." The fact that this is a stable string and not a
  composite source-and-name is important — it survives a hook moving
  between, say, project config and a plugin bundle, or being upgraded
  in-place by a plugin.

Discovery surface in `codex-rs/hooks/src/registry.rs` (and adjacent
`engine/discovery.rs`, `engine/dispatcher.rs`) extends the `HookSource`
enum to include `Plugin` — visible at
`codex-rs/analytics/src/events.rs:687` where the analytics mapping is
extended with `HookSource::Plugin => "plugin"`. New `codex-plugin`
dependency in the `codex-hooks` crate (Cargo.lock and
`codex-rs/hooks/Cargo.toml`) is the carrier; reverse — `codex-plugin`
also gets `codex-config` for the user-level `[[hooks.config]]`
persistence schema.

User-level disable persistence lands in `codex-rs/config/src/hook_config.rs`
(new file) plus an edit path in `codex-rs/core/src/config/edit.rs` and
its tests at `edit_tests.rs`. The split — discovery in `hooks/`,
persistence in `config/` — is the right layering: the hook engine
shouldn't have to know how user prefs are serialized, and the config
crate shouldn't have to know what makes a hook discoverable.

Server side: `codex-rs/app-server/src/codex_message_processor.rs`
gains `hooks/list` and `hooks/config/write` dispatchers (paralleling
the existing `skills/list`), with a focused integration test at
`codex-rs/app-server/tests/suite/v2/hooks_list.rs`.

## Risks / nits

1. **64 files changed, +2565/-236** — but the shape (schema additions
   + new file `hook_config.rs` + new dispatcher pair + analytics enum
   variant + plugin loader/manager touches) is consistent with the
   stated scope. No drive-by churn visible in the file list.
2. **"Stable hook keys"** are the load-bearing abstraction and the PR
   body doesn't define them. Are they `(source, name)` tuples?
   Plugin-id-prefixed? Hash of definition? The stability contract
   matters because the `[[hooks.config]]` user file persists keys
   forever and needs to gracefully handle "key no longer matches any
   discovered hook." A one-paragraph note on the format and
   forward/backward-compat policy belongs in the changelog.
3. **Disabled-but-listed semantics** need documenting in the
   `HooksListResponse` schema doc. The current TS/JSON schemas at
   `app-server-protocol/schema/typescript/v2/HooksListResponse.ts`
   and `HooksListEntry.ts` should explicitly describe the
   "disabled entries are returned for management; runtime suppresses
   them" contract so app-client authors don't have to grep the
   server source to learn it.
4. **Stale-config GC** — what happens when a user disables hook X
   from plugin P, then uninstalls plugin P? Does the entry persist
   forever in `[[hooks.config]]`? A periodic prune pass keyed off
   "last seen during discovery" would be safer; minimum, document
   the current behavior.
5. **Test coverage at the integration layer** is just `hooks_list.rs`
   (new). A `hooks_config_write` integration test pinning the
   round-trip — list → disable → list-shows-disabled → enable →
   list-shows-enabled, with a reload between steps to confirm
   persistence — would close the obvious gap.
6. **`HookErrorInfo`/`HookMetadata`/`HookSource` TypeScript types**
   are new files that app clients will need. Confirm the bundled
   TS-types package picks them up via `index.ts` re-export (visible
   at `app-server-protocol/schema/typescript/v2/index.ts` in the
   diff but worth eyeballing).

## Verdict

`merge-after-nits` — the layering is right, the protocol additions
parallel the existing `skills/*` surface (which is the project's
established pattern), and the analytics + persistence + dispatcher
+ test all land in the same PR which is correct for a coherent
feature. The "stable key" definition, the `HooksListResponse`
disabled-semantics doc, and a list↔write round-trip integration
test should land before merge.

## What I learned

When a feature has parallel "discovery + user-prefs override" shapes
(skills now, hooks here), the *protocol* should mirror across them
intentionally — clients learn the pattern once and reuse it. The
load-bearing abstraction is the "stable key" that survives a
discovery source changing; getting that wrong is the kind of bug
that's invisible at merge time and silently corrupts user prefs
six months later.
