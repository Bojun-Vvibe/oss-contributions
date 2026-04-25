# cline/cline PR #10331 — feat(ui): redesign chat toolbar with popup panels

@df35f5f4 · base `main` · +1314/-123 · author `raidengr40`

## Summary
Large UI refactor: removes the always-visible `AutoApproveBar` from the chat footer and consolidates context, file picker, auto-approve, and model selector controls into focused popup panels. Pure UI / UX changes per the PR body — no backend, proto, or hook changes claimed.

## What changed
- `webview-ui/src/components/chat/toolbar-popups/PlusPopup.tsx` (new) — `+` button popup with context/files section and an auto-approve accordion (expanded by default).
- `webview-ui/src/components/chat/toolbar-popups/AddContextModal.tsx` (new) — URL / Problems / Terminal / Git / Folder / File rows with single-open accordion behavior.
- `webview-ui/src/components/chat/toolbar-popups/ModelSelectorDropdown.tsx` (new) — inline scrollable model list.
- `webview-ui/src/components/chat/toolbar-popups/index.ts` (new) — barrel export.
- `webview-ui/src/components/chat/ChatView.tsx` — removes `<AutoApproveBar />` from footer.
- `webview-ui/src/components/chat/ChatTextArea.tsx` — bottom toolbar block (around lines 1532-1622 per the plan) reworked to host the new popups; `ModelDisplayButton`, `SwitchContainer`, `Slider` styled-components restyled.
- `webview-ui/src/index.css` & `theme.css` — new design-token CSS custom properties.
- `implementation_plan.md` (new, 194 lines) — design doc.

## Key observations
- `implementation_plan.md` should not be checked in to the repo root. It belongs in a docs/ folder or a PR description; otherwise it becomes stale tribal knowledge sitting next to source.
- The plan explicitly states no logic / API changes, but `src/core/hooks/__tests__/hook-factory.test.ts` is touched in the file list — that's outside webview-ui and contradicts the scope claim. Needs explanation.
- Removing `AutoApproveBar` from the footer is a discoverability regression risk: power users glance at it constantly. Consider an opt-in setting to restore it, or at minimum a one-time toast on first launch.
- Three new popup components share anchor/positioning logic (`anchorRef: React.RefObject<HTMLElement>`); extract a `useAnchoredPopover` hook to avoid drift.
- Toggle switches reimplemented as raw `<label class="toggle">` with hand-rolled `28×16px` CSS rather than reusing `VSCodeCheckbox`. Inconsistent with the rest of the extension's a11y story; screen readers will not get the right semantics without an explicit `role="switch"` and `aria-checked`.
- 1,314 added lines without a single new unit test. For a refactor of this size touching the input surface, snapshot/RTL tests on `ChatTextArea` are reasonable to require.
- Keyboard navigation between popups not addressed in the plan — focus trap + Esc-to-close should be verified.

## Risks/nits
- Disruption to muscle memory of existing users; warrants a CHANGELOG entry.
- Hard-coded colors (`#2563eb`, `#2e2e2e`) bypass the theme system; should reference theme tokens for VSCode dark/light/high-contrast parity.

**Verdict: request-changes**
