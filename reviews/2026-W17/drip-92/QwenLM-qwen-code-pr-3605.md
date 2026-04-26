---
pr: 3605
repo: QwenLM/qwen-code
sha: 870e940dc37076662e33f37fd8c20fd5c28d6645
verdict: merge-after-nits
date: 2026-04-27
---

# QwenLM/qwen-code #3605 — feat: adds a Space-to-preview affordance to the /resume session picker

- **Author**: qqqys
- **Head SHA**: 870e940dc37076662e33f37fd8c20fd5c28d6645
- **Size**: +766/-26 across 8 files. Net new: `SessionPreview.tsx` + tests, plus picker/hook plumbing.

## Scope

In the `/resume` picker, pressing **Space** on a row inline-renders that session's conversation through the real `HistoryItemDisplay` pipeline (the same renderer the live chat uses). Enter resumes, Esc returns to the list. New `enablePreview` opt-in prop guards the feature so other callers of `SessionPicker` (e.g. delete flows, where Enter is destructive) don't accidentally trigger "Enter to commit" through the preview.

## Specific findings

- `useSessionPicker.ts:98-104` — clean state model: `viewMode: 'list' | 'preview'` + `previewSessionId`, with `exitPreview` resetting both atomically via `useCallback`. The keypress handler at `:234-238` short-circuits when `viewMode !== 'list'` ("Preview owns the keyboard while active") and the `useKeypress` `isActive` prop at `:319` is `isActive && viewMode === 'list'` — belt-and-suspenders against double-handling. Good.
- `useSessionPicker.ts:291-298` — Space dispatch: `if (name === 'space' && enablePreview)`. Off-by-default opt-in, snapshots `filteredSessions[selectedIndex]` so a concurrent filter change can't crash. Correct.
- `SessionPicker.tsx:31-36` — `enablePreview?: boolean` prop with the **best comment in the PR**: explains why this is opt-in ("preview's Enter shortcut forwards to `onSelect`, which for resume flows is 'resume', but for destructive flows (e.g. delete) would commit the action"). This is exactly the kind of foot-gun documentation that prevents future regressions.
- `SessionPicker.tsx:174-191` — preview branch returns the `SessionPreview` component directly when `viewMode === 'preview' && previewSessionId`. The `previewed = picker.filteredSessions.find(s => s.sessionId === picker.previewSessionId)` lookup at `:179-181` is O(n) per render — fine at typical session counts (<200) but worth a `useMemo` if anyone ever loads thousands.
- `SessionPreview.tsx:331-353` — load lifecycle uses `cancelled` flag in the cleanup callback to avoid `setState` after unmount when `loadSession` resolves late. Standard React pattern, correct here.
- `SessionPreview.tsx:357-360` — comment "Preview passes `null` config: tool_group entries degrade to name-only (no description). Users can press Enter to resume for full fidelity." This is a deliberate trade-off worth flagging in the PR body — "preview is lossy for tool_group rows" is user-visible behaviour.
- `SessionPreview.tsx:380-383` — narrow-terminal hardening: `boxWidth = Math.max(10, columns - 4)` and `separatorWidth = Math.max(0, boxWidth - 2)`. Comment explicitly calls out the `'─'.repeat(n)` `RangeError` failure mode at narrow widths. Excellent.
- `SessionPreview.tsx:365-378` — keypress handler distinguishes Esc/Ctrl+C (`onExit`) from Enter (`onResume(sessionId)`). No allowance for any other key, so navigation inside the preview (PgDn, arrows) doesn't scroll — relies on the terminal's own scrollback per the `Body` comment "let the terminal's scrollback own overflow". Documented choice, fine.
- Tests: `SessionPreview.test.tsx` (loading, error, metadata footer, branch line, item count), `StandaloneSessionPicker.test.tsx` (Space opens preview, Esc closes, no-`enablePreview` is no-op, footer hint conditional). Solid coverage of the surface.
- Missing test: behaviour when `previewSessionId` references a session that's been filter-removed during preview (concurrent filter input from a re-render). The `find` returns undefined and component falls back to `sessionTitle ?? t('Session Preview')` — graceful, but worth a one-line test.

## Risk

Low. Opt-in via `enablePreview`, doesn't touch the resume code path, defensive against narrow terminals and async-load races. The "tool_group rows degrade to name-only" behaviour is documented in code but should make it into release notes.

## Verdict

**merge-after-nits** — the design is solid (keyboard ownership, opt-in to protect destructive flows, narrow-terminal clamps). Worth tightening: (1) document the lossy `null`-config preview behaviour in the PR body / changelog, (2) add a regression test for "preview-target gets filtered out during preview", (3) consider memoizing the `filteredSessions.find` lookup. None of these block merge.
