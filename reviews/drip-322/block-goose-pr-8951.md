# block/goose PR #8951 — settings page polish + About metadata

- Link: https://github.com/block/goose/pull/8951
- SHA: `8171fa67b42427cf6963626c5c3e651ff3394f34`
- Author: delkc
- Stats: +135 / −70, ~10 files under `ui/goose2/src/features/settings/ui/`

## Summary

Removes the now-redundant per-tab description copy from goose2 Settings pages (the shared header line covers it), keeps page-level header actions colour-aligned with the standard outline button, refines the auto-compaction copy, and — the substantive change — populates the About page with real app metadata: name, version, build mode, Tauri version, identifier, and a literal `Apache-2.0` license row.

## Specific references

- `ui/goose2/src/features/settings/ui/AboutSettings.tsx` L1–L60: lazy `import("@tauri-apps/api/app")` is gated behind a `window.__TAURI_INTERNALS__` check. Correct — when the page is rendered outside Tauri (e.g. Storybook, vitest jsdom), the dynamic import is skipped instead of throwing. The cancellation flag (`let cancelled = false; … return () => { cancelled = true }`) handles strict-mode double-mount cleanly.
- L40–L48: `Promise.all([getName(), getVersion(), getTauriVersion(), getIdentifier()])` — good, four independent Tauri RPCs in parallel instead of sequential awaits.
- L49–L55: the `catch { setAppInfo(null) }` swallows the error silently. Two improvements: (1) include a `console.warn` (or use the existing logger) so a Tauri RPC regression doesn't disappear; (2) consider distinguishing "not in Tauri" from "Tauri RPC failed" — currently both render the i18n `about.unavailable` fallback string. A small `errored` boolean would let the UI show a clearer state.
- L66–L100: `AboutInfoRow` rows fallback to `appInfo?.name ?? "Goose"` for name (hard-coded default) but `?? fallback` (i18n string) for everything else. Slightly inconsistent — the name fallback should also go through i18n so non-English builds don't render the bare English word.
- L99: `<AboutInfoRow label={t("about.fields.license")} value="Apache-2.0" />` — license string is hard-coded. Acceptable for the project's actual license, but if the license ever changes (sub-licensed vendor builds, etc.) this is silent. Consider sourcing from `package.json` via `import.meta.env` build-time injection.
- `ExtensionsSettings.tsx` L143: the `description={t("extensions.description")}` prop is dropped. Confirmed consistent with the "shared header line is in place" rationale — assuming the SettingsPage shell now renders the description from a shared source, this is correct. If not, it's a regression.
- `CompactionSettings.tsx`, `GeneralSettings.tsx`, `ProjectsSettings.tsx`, `VoiceInputSettings.tsx`, `DoctorSettings.tsx`, `ChatsSettings.tsx`, `ExtensionsSettings.tsx` all drop `description=` props and minor copy. Wide cross-cutting touch — easy to miss one and end up with one tab still rendering double headers. Worth a quick visual sweep of every settings tab in the dev app, which the PR body says is "ready for visual review in the running goose2 dev app".

## Verdict

verdict: merge-after-nits

## Reasoning

The About-page fetch logic is implemented correctly (Tauri-internals check, `Promise.all`, strict-mode cleanup). The settings-tab descriptive-copy cleanup is mechanically simple but cross-cutting — it depends entirely on the shared header actually rendering the description, which isn't visible in this diff. Nits: log Tauri RPC failures, route the "Goose" name fallback through i18n, sample at least one settings tab visually before merge. Test coverage listed (`pnpm check`, `pnpm typecheck`, the targeted `GooseAutoCompactSettings` test) is appropriate.
