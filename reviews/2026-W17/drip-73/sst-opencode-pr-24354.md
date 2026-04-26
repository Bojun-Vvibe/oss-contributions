---
pr: 24354
repo: sst/opencode
sha: 7fdc0d3192232602cc8e23f67ecc6e3462124337
verdict: request-changes
date: 2026-04-26
---

# sst/opencode #24354 — feat(app): add confirmation dialog before archiving session

- **Author**: herjarsa
- **Head SHA**: 7fdc0d3192232602cc8e23f67ecc6e3462124337
- **Size**: 6,426 diff lines across 60+ files; the *actual* feature is ~80 lines in 4 files.

## Scope

The advertised change is a `DialogArchiveSession` confirmation dialog mirroring the existing `DialogDeleteSession` pattern, wired into the layout command (`packages/app/src/pages/layout.tsx`) and the message-timeline dropdown (`packages/app/src/pages/session/message-timeline.tsx`). It also refactors `archiveSession` into `executeArchiveSession(session)` plus a thin `archiveSession(sessionID)` wrapper so both call sites can reach the same dialog. There are two unrelated drive-bys: a `pty.created` event subscription in `packages/app/src/context/terminal.tsx` and an "auto-open terminal panel when agent creates a PTY" effect in `packages/app/src/pages/session/terminal-panel.tsx`.

## Specific findings

- **Diff hygiene is broken.** The patch ships *thousands* of lines of personal tooling cruft that has nothing to do with the feature: `.atl/skill-registry.md`, `.claude/memory/github-pr-template-guide.md`, four `.playwright-mcp/` console+page captures (~3,500 lines), `.pr-body-24287.md`, `.pr-body-6527-template.md`, `.pr-body-6527.md`, `.pr-body.txt`, `.sisyphus/drafts/*`, `.sisyphus/plans/*`, `.sisyphus/ralph-loop.local.md`, `.tmp-issue-body.md`, `.tmp-pr-body.md`, `.typecheck-results.md`, `COMMIT_MSG.txt`, `check_comments.py`, plus a duplicate `packages/opencode/.typecheck-results.md`. These are all author-machine working files. The repo's `.gitignore` should exclude these globs, and the PR must be rebased to drop them — they will conflict on every other contributor's branch and leak the author's local agent setup into the project tree. This single issue is verdict-changing.
- **`packages/app/src/pages/layout.tsx:989` — split is correct.** Renaming the original `archiveSession(session)` to `executeArchiveSession` and re-introducing `archiveSession(sessionID)` as a thin lookup-then-execute wrapper is the right shape so the `command.register("layout", …)` path (which already had a `session` object) and the message-timeline path (which only had an ID) can share the new dialog. But there's no test for either path.
- **`packages/app/src/pages/layout.tsx:1107`** — the command path now opens the dialog directly (`dialog.show(() => <DialogArchiveSession session={session} />)`) instead of calling `archiveSession`. Good, but this means the command can no longer be invoked headlessly (e.g., from a keybinding script) without UI confirmation. If that is intentional (consistent with delete), say so in the PR body. If not, add a `force` variant or a Shift-modifier bypass like delete uses.
- **`packages/app/src/pages/session/message-timeline.tsx:626` — the second `DialogArchiveSession` is a duplicate definition.** It takes `{sessionID, title}` instead of `{session}` because the timeline dropdown only has the id. The dialog body is otherwise byte-identical to the layout copy. Extract a shared `<DialogArchiveSession>` (in `packages/app/src/components/` or alongside `DialogDeleteSession`) that accepts `{title, onConfirm}` so the two callers reduce to props.
- **`packages/app/src/pages/session/message-timeline.tsx:910`** — the dropdown handler now reads `sync.session.get(id)?.title` synchronously to compute the dialog title, falling back to `language.t("command.session.new")`. Same fallback logic exists in the layout copy via `sessionTitle(props.session.title)`. The two title-derivation paths can drift; centralize.
- **Unrelated drive-bys, `packages/app/src/context/terminal.tsx:188-200` and `packages/app/src/pages/session/terminal-panel.tsx:68-80`** — the `pty.created` subscription and the auto-open-terminal effect are real fixes (without the subscription, `terminal.all()` never grows when the agent creates a PTY, so the auto-open effect would never fire), but they have nothing to do with the archive dialog and should be a separate PR with its own description, especially since the auto-open-on-`source==="agent"` behavior is a UX policy decision that deserves its own discussion.
- **i18n keys `session.archive.title`, `session.archive.confirm`, `session.archive.button`** — referenced but the patch slice does not show the corresponding `language` resource additions. Either they're elsewhere in the diff (likely buried under the cruft) or this will render `[missing key]` strings in non-default locales. Verify all 8 locales have entries.
- **No tests.** Both `DialogDeleteSession` and the new `DialogArchiveSession` are pure UI; at minimum, a smoke test that confirms the dropdown's `onSelect` path opens a dialog and the dialog's primary button calls `executeArchiveSession` (mocked) would prevent the next refactor from regressing the wiring.

## Risk

High **process** risk, low **change** risk. The actual feature is sound and matches the existing delete pattern. The 6,000+ lines of personal-tooling noise is unacceptable as-is — it pollutes `git log`, complicates rebases for everyone else, and leaks author's `.claude/`, `.atl/`, `.playwright-mcp/`, `.sisyphus/` workflows into the canonical tree. Merging as-is would normalize this pattern.

## Verdict

**request-changes** — (1) drop every non-`packages/` file from the diff and add the corresponding entries to `.gitignore`; (2) split the `pty.created` + auto-open-terminal-on-agent-PTY changes into a separate PR; (3) extract a single shared `DialogArchiveSession` component instead of two copies; (4) confirm i18n key coverage; (5) add a smoke test for the dropdown→dialog→execute path. The feature itself is fine; the PR is not.
