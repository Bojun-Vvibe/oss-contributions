# anomalyco/opencode #24251 — Add project run configs to the web app

- **Repo**: anomalyco/opencode
- **PR**: [#24251](https://github.com/anomalyco/opencode/pull/24251)
- **Head SHA**: `3af665c3b82710b08bd9e34b3c59a389c124dcd4`
- **Author**: atlantis-mk
- **State**: OPEN
- **Verdict**: `needs-discussion`

## Context

Adds a "Run Configs" feature to the web app surface: per-project
detect-and-execute presets (like an IDE's run configurations) sourced from
package.json scripts, Makefile targets, and Cargo/Maven/Gradle/Go entry
points. Adds a new dialog
(`packages/app/src/components/dialog-edit-run-configs.tsx`), a
detection/scaffolding library (`packages/app/src/pages/session/run-config.ts`,
291 lines new), session-header wiring, store/sync plumbing, terminal
integration, and trilingual i18n entries. ~1.1k additions across 19 files.

## Design

The detection module at `run-config.ts:25-55` is the heart of the change.
It defines a denylist of `IGNORED_PROJECT_DIRS` (`.git`, `node_modules`,
`dist`, `target`, etc.) and a positive `MARKER_FILES` set
(`package.json`, `go.mod`, `pom.xml`, `Cargo.toml`, `Makefile`,
gradle/maven wrappers). The walker then constructs `ProjectCandidate`
objects per directory containing markers and emits `RunConfig` rows
scoped via `scopedRunConfigs` at `run-config.ts:71-78` so monorepos get
`packages/foo: dev` style titles. Sensible.

`packageRunConfigs` at `run-config.ts:81-99` correctly handles the
`packageManager` field (`bun`/`pnpm`/`yarn`/`npm`) and emits one config
per script — though see Risks #2.

The store wiring touches `global-sync/child-store.ts:157-220` and
`global-sync/types.ts:22-34` to add `runConfigs` as a synced collection,
and the session-header dialog edit dance is plumbed via
`session/session-header.tsx:269-388` (the new ~100-line popover block).
Test coverage exists at `pages/session/run-config.test.ts` (~180 lines)
exercising package detection, Makefile parsing, and the monorepo
scope-prefixing. Good.

## Risks / Nits

1. **Detection-time arbitrary-command surface.** `run-config.ts` builds
   a `command` string from each detected `package.json` script verbatim
   — `npm run <name>` where `<name>` is whatever the project author put
   in `pkg.scripts`. The dialog at
   `dialog-edit-run-configs.tsx:181` then offers a one-click "run"
   button that pipes this into the terminal context
   (`context/terminal.tsx:119-211` changes). On a freshly cloned untrusted
   repo, "scan" + "run" becomes a single-click RCE surface against the
   user's machine. The current per-script `runConfig` invocation at
   `dialog-edit-run-configs.tsx:198-208` doesn't gate behind a
   confirmation prompt for newly-detected (vs user-edited) configs.
   Consider: scanned configs should be in a "review" state until the
   user explicitly accepts them, separate from manually authored ones.

2. **Common-target Makefile heuristic is too eager.** The
   `COMMON_MAKE_TARGETS` set at `run-config.ts:17` is
   `{dev, run, start, serve, test, build}`. The detection logic (further
   in the file) presumably emits a config for every Makefile target whose
   name appears in this set, regardless of what the target actually does.
   In a typical infra repo, `make build` might invoke Docker, push an
   image, and require credentials. Surfacing it as a one-click "run"
   without showing the recipe body is a footgun.

3. **`ProjectCandidate.files` is a `Set<string>`** at `run-config.ts:60`
   and the test file at `run-config.test.ts` uses similar shapes, but
   the sync layer in `child-store.ts:157` serializes `runConfigs` —
   confirm the Set isn't accidentally serialized through the IPC bridge,
   which would silently drop entries on JSON round-trip. The PR does not
   show a serializer.

4. **i18n drift.** `i18n/en.ts`, `zh.ts`, `zht.ts` all gain
   `dialog.project.edit.runConfigs.*` keys at lines ~673-810. The
   English copy says "scanning"/"add"/"remove" but the zh/zht
   translations should be spot-checked by a native speaker — this PR
   author appears to have machine-translated based on the punctuation
   patterns. Not blocking but worth a follow-up label.

5. **No daemonization story.** A run-config that starts a long-lived
   server (`npm run dev`) holds the terminal session. The terminal
   context changes at `context/terminal.tsx:148-211` add some workspace
   bookkeeping but I don't see explicit handling for the
   "config-launched-this-pty" lifecycle: e.g., on session close, are the
   spawned PTYs killed? On config edit, does the running command get
   re-launched or left orphaned?

## What I learned

This is the right shape for a feature (detect → preview → execute) but
the security model isn't fully specified. The interesting design tension:
auto-detection makes the feature delightful for trusted repos but turns
the agent into an arbitrary-command executor for untrusted ones. The
mitigation is a per-config "trust" bit gated by user action, not a
denylist of "scary" commands — denylists never close. Recommend pairing
this PR with an explicit review of the trust boundary before merge.
