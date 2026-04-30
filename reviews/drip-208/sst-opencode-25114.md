# sst/opencode#25114 — fix(desktop): Prevent Model response Interruption when opening settings dialog

- PR: https://github.com/sst/opencode/pull/25114
- Head SHA: `31d821ee7856fc0b666b20760106cbf58ce7ea7b`
- Author: OpeOginni
- Closes: #24859
- Files: 2 changed, +9 / −3

## Context

Opening the Settings dialog interrupted any in-flight agent turn with `MessageAbortedError`. Root cause chain: `<SettingsGeneral />` mounts → `createResource(() => globalSdk.client.pty.shells())` resolves → `shellOptions()` rebuilds → Kobalte `<Select>` sees a new `current` reference and fires `onChange` with the auto option (value `""`) → `globalSync.updateConfig({ shell: "" })` → `PATCH /global/config` → `Config.invalidate()` → `Instance.disposeAll()` tears down every running fiber. The teardown happens with no `/session/abort` involvement, so the user just sees responses vanish.

## Design

Two complementary fixes, one client-side, one server-side defense in depth:

1. **Client (settings-general.tsx:332)** — `onSelect` now bails when `option.value === currentShell()`:
   ```ts
   if (option.value === currentShell()) return
   globalSync.updateConfig({ shell: option.value })
   ```
   Stops the spurious write at the source (the spurious mount-time onChange).

2. **Server (config/config.ts:759-779)** — `Config.updateGlobal` now diffs serialized config against the on-disk bytes and only writes + calls `invalidate()` when the content actually changed. Both the JSON and JSONC branches are covered:
   ```ts
   const serialized = JSON.stringify(merged, null, 2)
   changed = serialized !== before
   if (changed) yield* fs.writeFileString(file, serialized).pipe(Effect.orDie)
   ```
   then `if (changed) yield* invalidate()` at the bottom. This protects against any other client (TUI, future surfaces, third-party SDK consumers) that does the same no-op `updateConfig`.

## Risks / nits

- The diff is keyed on serialized-string equality, which is sensitive to key ordering and whitespace. For the JSON branch that's fine because `JSON.stringify(merged, null, 2)` is deterministic and the stored file came from the same serializer. For the JSONC branch, `patchJsonc` preserves the original formatting, so equality on raw `updated !== before` is the right semantic — confirmed in the diff. Good.
- Layered defense is appropriate because the same bug class can show up on any new settings panel that mounts a resource and rebinds a Kobalte `Select`. The client-level guard fixes today's bug; the server-level guard makes the next variant a no-op instead of a regression.
- No new test added for the server-side diff-and-skip path; would be a useful regression test against a future refactor that re-introduces unconditional invalidation. Not blocking — the behavior is small and the body-described electron build verification covers the user-visible case.

## Verdict

**merge-as-is**
