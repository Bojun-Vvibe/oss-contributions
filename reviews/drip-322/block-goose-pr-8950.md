# block/goose PR #8950 — settings modal: shared SettingsPage shell + compact pinned headers

- Link: https://github.com/block/goose/pull/8950
- SHA: `233e83b576b48feb09d466f873766d7d38f9a264`
- Author: kalvinnchau
- Stats: +452 / ~, ~12 files across `ui/goose2/src/features/{settings,extensions}/ui/` and `ui/goose2/src/shared/ui/`

## Summary

Standardises the goose2 settings-modal chrome by introducing a shared `SettingsPage` wrapper with `title`, `description`, `actions`, `controls`, and a children content area. Settings sections (Extensions, Appearance, Chats, Compaction, Doctor, General, About, Voice Input, Projects) are migrated to the shell, and the Extensions search field is converted from a hand-rolled `<Input>+<IconSearch>` to the shared `SearchBar` primitive. Doctor moves its actions into the compact header. About is also extracted to its own component to match other pages.

## Specific references

- `ui/goose2/src/features/extensions/ui/ExtensionsSettings.tsx` L141–L197: the migration to `SettingsPage` collapses ~25 lines of bespoke header+search markup into props. Good. The `actions={...}` slot now hosts the "Add Extension" button — note `size="xxs"` (header-button size) replaces the previous bottom-of-page `size="sm"` button, *and* the bottom-of-page button is removed entirely (L210–L222 in the old file). That's a real UX change (button moves from bottom to top header), not just a refactor — worth confirming with design that this is intentional across all sizes/screens.
- L156–L165: `<SearchBar value={searchTerm} onChange={setSearchTerm} … size="compact" />` replaces the `<Input>` with `onChange={(e) => setSearchTerm(e.target.value)}`. The new shared component takes the value directly instead of an event — cleaner API, but make sure consumers in tests aren't still firing synthetic events.
- L196–L197: `</SettingsPage>` closes around the `{modalMode === "add" && <ExtensionModal …/>}` and the edit-modal block. Putting modals *inside* the SettingsPage wrapper is unusual — modals typically render via portal anyway, but if `SettingsPage` adds `overflow-hidden` or any clipping, the modal animation could clip on first frame. Worth a quick visual check.
- `AboutSettings.tsx` (new file in this PR, then expanded by the sibling PR #8951): this PR introduces the bare scaffold (`title`, `description`, no body). PR #8951 then fills in the actual app-metadata rows. The two PRs are clearly designed to land together — if #8951 lands first the About tab is empty, if #8950 lands first the About tab is just a header. Either order is recoverable but reviewers should sequence them.
- The cross-cutting nature (every settings tab touched) means one missed file is easy to spot post-merge but hard to spot pre-merge. PR body lists `ExtensionsSettings`, `AboutSettings`, `AppearanceSettings`, `ChatsSettings`, `CompactionSettings`, `DoctorSettings`, `GeneralSettings`, `ProjectsSettings`, `VoiceInputSettings` — that matches the file list. The `__tests__/GooseAutoCompactSettings.test.tsx` is also touched, suggesting the test was updated to match the new wrapper.
- The shared `SearchBar` and `SettingsPage` primitives are not shown in the diff hunk captured here; they should land or already exist in `ui/goose2/src/shared/ui/`. If they're new in this PR, they're the most important files to review (they define the contract every other tab now depends on). If they pre-exist, this PR is purely consumer-side.

## Verdict

verdict: merge-after-nits

## Reasoning

Mechanically clean settings-shell consolidation. The `actions={...}` API and `SearchBar` migration are real ergonomics wins for future settings tabs. Nits before merge: (1) confirm the Extensions "Add Extension" button moving from bottom to top header was design-approved (it's a UX change, not a refactor); (2) sequence with PR #8951 explicitly so About isn't briefly empty in main; (3) verify modals rendered inside `SettingsPage` aren't clipped by any wrapper overflow rules. The shared primitives are the real contract — make sure those are reviewed even if they're not in this exact diff range.
