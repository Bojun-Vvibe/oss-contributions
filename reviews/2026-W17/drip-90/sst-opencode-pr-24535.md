---
pr: 24535
repo: sst/opencode
sha: ed17baa12ec881412a223f4492dfd808e1626184
verdict: needs-discussion
date: 2026-04-27
---

# sst/opencode #24535 — feat: add multi-root workspaces

- **Head SHA**: `ed17baa12ec881412a223f4492dfd808e1626184`
- **Author**: cgarrot
- **Size**: large feature (new `WorkspaceProvider`, `dialog-workspace-create.tsx`, `e2e/workspace.spec.ts`, plus edits to `app.tsx`, `dialog-select-file.tsx`)

## Summary
Introduces a multi-root workspace concept to the desktop app: a `WorkspaceProvider` context, a "Create Workspace" dialog that lets the user name a workspace and add N folder paths, an "Open Workspace" command, plus Playwright e2e coverage. Folder selection is delegated to the existing `DialogSelectDirectory`.

## Specific findings
- `packages/app/src/app.tsx:99-101` — `WorkspaceProvider` is wrapped *inside* `HighlightsProvider` and around `Layout`. Reasonable, but downstream providers like `ServerProvider`/`ModelsProvider` are still keyed off a single root assumption — those need either workspace-aware re-keying or an explicit ADR explaining "first folder is the canonical project root, others are read-only siblings". The PR adds the surface but doesn't address the semantic question.
- `packages/app/src/components/dialog-workspace-create.tsx:23-31` — the `once()` wrapper is a one-shot guard around dialog callbacks. Correct pattern (prevents both the directory `onSelect` and the `onClose` from firing the reopen path twice), but the closure captures `currentName` / `currentFolders` *at the time `addFolder()` is called*. If the user types in the name field while the directory picker is open, the typed value is silently discarded on reopen. Either snapshot inside the picker callback or store name/folders in a ref the callback reads on fire.
- `packages/app/src/components/dialog-workspace-create.tsx:56-60` — `nextFolders` dedupe uses `currentFolders.includes(normalized)` after `result.trim()`. No path normalization (no `path.resolve`, no trailing-slash stripping, no case-folding on Windows). User can add `~/foo` and `~/foo/` and get two entries pointing at the same dir, which then explodes downstream when the workspace is opened.
- `packages/app/src/components/dialog-workspace-create.tsx:81-92` — `handleCreate()` catches `err` and renders `err.message` in `.text-text-error`, but never clears `error` when the user edits inputs again after the failure. Add `setStore("error", "")` to the input `onChange` handlers (already done for the name field, missing for the folder list mutations).
- `packages/app/src/components/dialog-select-file.tsx:43-44` — adds `workspace.open` and `workspace.create` to `COMMON_COMMAND_IDS` *and keeps* `workspace.new`. If `workspace.new` is being deprecated by `workspace.create`, ship the removal in the same PR or document the relationship in the PR body — otherwise users will see two commands that look identical.
- `packages/app/e2e/workspace.spec.ts:29-32` — `addFolderFromDialog` types the absolute path into the dialog input and presses Enter. Real users hit `DialogSelectDirectory`, which presumably has a fuzzy picker UI. The e2e tests therefore verify the *string-input* code path but never the actual directory-picker integration that `addFolder()` wires up. Add at least one test that drives the picker by keystrokes.
- `packages/app/e2e/workspace.spec.ts:40-65` — single test case asserts `Recent projects` is visible in `beforeEach`; if a previous test pollutes the recent-projects list with a name collision, `await expect(page.getByText(workspaceName)).toBeVisible()` becomes flaky. The `Date.now()` suffix mitigates but doesn't eliminate (machine clock can rewind in CI containers using `date --set`).

## Risk
Medium. Net-new surface, but the "first folder = root" question and the dedupe gap will cause confusing bug reports once users start adding folders by drag-drop from Finder (which yields trailing slashes).

## Verdict
**needs-discussion** — the dedupe normalization, the `workspace.new` vs `workspace.create` overlap, and the "what does multi-root actually mean for the existing providers" question all need a maintainer answer before this lands.
