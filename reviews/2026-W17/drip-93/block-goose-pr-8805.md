---
pr: 8805
repo: block/goose
sha: 4924018b78a7cbbc84a817d27d0c8d34f36a0681
verdict: merge-after-nits
date: 2026-04-27
---

# block/goose #8805 — [RFC] feat(ui): group chat sessions by project

- **Author**: vincenzopalazzo
- **Head SHA**: 4924018b78a7cbbc84a817d27d0c8d34f36a0681
- **Link**: https://github.com/block/goose/pull/8805
- **Size**: +546/-128 across the desktop UI, with a new `src/utils/projectSessions.ts`, a 150-line unit-test file, and prop-plumbing through `ChatSessionsDropdown`, `SessionsList`, `CondensedRenderer`, `ExpandedRenderer`, and `NavigationPanel`.

## Scope

The session list in the desktop UI was a flat newest-first list. This PR groups sessions by their `working_dir` so that — when sessions span multiple projects — the dropdown and sidebar render `ProjectGroup → Session[]` instead of one long list. Also extracts a `resolveNewChatWorkingDir()` helper that picks the active session's working dir for "New Chat" with a sensible fallback chain.

## Specific findings

- `src/utils/projectSessions.ts` (new) — three exports tested directly: `groupSessionsByProject`, `getProjectLabel`, `resolveNewChatWorkingDir`. Behaviour locked down by the new test file at `__tests__/projectSessions.test.ts:1-150`.
- `__tests__/projectSessions.test.ts:42-54` — `'canonicalizes trailing separators when grouping projects'` covers `'/tmp/goose'`, `'/tmp/goose/'`, and `'  /tmp/goose//  '` all collapsing to the same group with `path === '/tmp/goose'`. Solid normalization.
- `__tests__/projectSessions.test.ts:56-65` — empty / whitespace `working_dir` becomes a single group with `label: 'Unknown'` and `path: ''`. Reasonable defensive default.
- `__tests__/projectSessions.test.ts:71-78` — `'disambiguates projects with the same basename'` test: `/Users/me/work/goose` vs `/Users/me/forks/goose` get labels `'work/goose'` and `'forks/goose'` respectively. Good — basename-only labelling would have been a real-world UX trap.
- `__tests__/projectSessions.test.ts:18-28` — group ordering test asserts groups sort by **most-recent session within group** (newest-group-first). This is correct UX (you want the active project at the top) but it's worth noting: a project that hasn't been touched in a year can jump to the top of the list the moment one of its sessions gets a `updated_at` bump. Acceptable trade-off.
- `__tests__/projectSessions.test.ts:30-39` — within-group sort is also newest-first by `updated_at`. Consistent.
- `__tests__/projectSessions.test.ts:97-105` — `getProjectLabel('C:\\Users\\me\\goose') === 'goose'` covers Windows-style paths. Worth confirming this isn't only matching backslash literally — i.e. the label-extractor handles both separators.
- `__tests__/projectSessions.test.ts:118-148` — `resolveNewChatWorkingDir()` test covers four paths: active session found → its dir; no active id → fallback; missing id → fallback; blank dir → fallback. Trims whitespace. Comprehensive.
- `Layout/CondensedRenderer.tsx:25,134,244,316`, `Layout/ExpandedRenderer.tsx:19,210`, `Layout/NavigationPanel.tsx:46,184` — `recentSessionsByProject` prop threaded through the existing `recentSessions` plumbing alongside, not replacing. Means flat-list consumers continue to work and the grouped consumer can opt in. Reasonable migration approach but **leaves two parallel data sources for the same content**, which is a maintenance hazard. A future cleanup pass should make the grouped form the canonical one and derive flat-list views from it.
- `ChatSessionsDropdown.tsx:53-54` — `const groupedSessions = projectGroups.length > 1 ? projectGroups : null` — only switches to grouped UI when there's actually more than one project. Sensible UX (one-project users see no header chrome).
- `ChatSessionsDropdown.tsx:55-83` — extracted `renderSessionItem(session)` so the same row JSX renders in both the flat and grouped paths. Good DRY discipline.
- The PR is marked `[RFC]` in the title, suggesting the author wants design review. The design itself looks fine; the main thing to push back on is the dual-data-source plumbing.

## Risks

- **Two parallel data shapes** for the same session list (`recentSessions` and `recentSessionsByProject`) propagate through three renderers and the dropdown component. Any future feature that touches "recent sessions" has to remember to update both, and a desync will silently break either the flat fallback or the grouped path. A follow-up that makes the grouped form canonical and derives the flat list via `groups.flatMap(g => g.sessions)` would clean this up.
- The tests are unit-level only. No integration test exercises the renderers actually rendering grouped sections, so a typo in the new `<DropdownMenuLabel>` rendering path would slip through.

## Verdict

**merge-after-nits** — the grouping logic itself is well-tested (Windows paths, trailing separators, blank dirs, basename collisions all covered) and the feature is a clear UX improvement. But the dual-`recentSessions`/`recentSessionsByProject` plumbing should be flagged for a near-term consolidation pass, and at least one renderer-level integration test (group header renders, sessions appear in the right group) would meaningfully de-risk the UI side.
