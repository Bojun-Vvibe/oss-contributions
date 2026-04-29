# sst/opencode PR #24869 — feat: make it easier to toggle on/off paste summary in the tui

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/24869
- Head SHA: `d7f10c7349f75079de05607e19dac71cb75a8865`
- State: OPEN, +17/-1 across 2 files

## What it does

Promotes the existing `experimental.disable_paste_summary` config knob (file-level, requires editing `opencode.json`) to a per-user TUI toggle backed by `kv` storage at key `paste_summary_enabled`, exposed via the System command palette as "Disable/Enable paste summary". The user-visible default is preserved by layering: the initial signal value is `kv.get("paste_summary_enabled") ?? !sync.data.config.experimental?.disable_paste_summary`, so a user who previously opted out via config still sees the feature off until they opt in via the palette.

## Specific reads

- `app.tsx:301-303` — new signal:
  ```ts
  const [pasteSummaryEnabled, setPasteSummaryEnabled] = createSignal(
    kv.get("paste_summary_enabled") ?? !sync.data.config.experimental?.disable_paste_summary,
  )
  ```
  The `??` fallback is the load-bearing piece — it correctly distinguishes "user has not interacted with the toggle" (`undefined`) from "user explicitly disabled it" (`false`). Without `??` (e.g. `kv.get("...", true)`), the config-derived default would only apply on first launch, never on a fresh kv store.

- `prompt/index.tsx:1209-1214` — the actual gate flips from the legacy config to the kv:
  ```ts
  if (
    (lineCount >= 3 || pastedContent.length > 150) &&
    kv.get("paste_summary_enabled", true)
  ) {
    pasteText(pastedContent, `[Pasted ~${lineCount} lines]`)
  ```
  Note the **inconsistency between the read sites**: the signal initializer at `app.tsx:301` uses `kv.get("paste_summary_enabled")` (no default → falls through to `??`) while the gate at `prompt/index.tsx:1213` uses `kv.get("paste_summary_enabled", true)` (default `true`). For the very first paste in a session where the user has `experimental.disable_paste_summary: true` in config, the gate will read `true` (palette item title will say "Disable paste summary"), but if the user pastes without ever opening the palette, the paste **will be summarized**, contradicting their config. The signal is computed but never *written* back to kv at startup.

- The companion palette entry at `app.tsx:739-751` is the correct shape: title computed from `pasteSummaryEnabled()`, `onSelect` flips and persists with `kv.set`. Pattern-matches the existing `terminal_title_enabled` toggle two blocks up.

## Verdict: `merge-after-nits`

## Rationale

The feature itself is straightforward and the kv-backed shape is the right primitive for a per-user TUI preference. But the two-site asymmetry between the initializer (`??` against legacy config) and the gate (`kv.get(..., true)` ignoring legacy config) means users who only ever set `experimental.disable_paste_summary: true` and never touch the palette will silently get summaries again on the very first paste — exactly the case the legacy config exists to handle. Two minimal fixes either (a) write the resolved initial value back to kv synchronously at startup so the gate site sees the effective default, or (b) change the gate to also consult the same `?? !sync.data.config.experimental?.disable_paste_summary` chain. Beyond that the changes pattern-match cleanly with `terminal_title_enabled` and the diff is small enough that one round of review covers it.
