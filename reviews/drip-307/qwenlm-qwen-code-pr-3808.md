# Review: QwenLM/qwen-code #3808 — docs(core): point background-shell guidance at both /tasks and the dialog

- **Repo**: QwenLM/qwen-code
- **PR**: #3808
- **Head SHA**: `5249601cc02cfed47d0a62321f57195b75ed1d3b`
- **Author**: wenshao

## What it does

Pure docs / LLM-facing string update. Follow-up to #3801. Updates two
LLM-visible tool-result strings (`shell.ts` after a background spawn,
`task-stop.ts` after a cancel request) and four internal comments /
docstrings to mention both inspection surfaces: the `/tasks` slash
command (text) and the interactive Background tasks dialog (footer
pill + Enter). Pre-#3488/#3720/#3791, only `/tasks` existed; the dialog
landed in those PRs and the strings were never updated.

## Diff notes

- `packages/core/src/tools/shell.ts:865` — replaces the single-surface
  hint with a two-surface hint that names the dialog explicitly and
  preserves the "read the output file" escape hatch. Length grew but
  this string is only emitted once per background spawn, so the
  per-call token cost is negligible.
- `packages/core/src/tools/task-stop.ts:110` and the adjacent comment
  at `:95` — symmetric treatment for the cancel path. Comment update at
  :95 is now technically accurate (the dialog *and* `/tasks` *and* the
  output file are all observable surfaces).
- `packages/core/src/services/monitorRegistry.ts:197` — fixes a real
  misnomer: the old comment said `"/tasks dialog"` which conflated the
  slash command with the overlay. New wording names them as separate
  surfaces. This is the highest-value change in the PR.
- `packages/core/src/services/backgroundShellRegistry.ts:10` (module
  docstring) and `:31` (`shellId` JSDoc) — adds the dialog to the list
  of consumers. Aligns the type-level documentation with the actual
  code paths.

## Concerns

1. The PR description acknowledges scope creep (committed to 2 sites,
   shipped 6) but the additional 4 are all single-line comment fixes
   with zero behavior risk. No issue.
2. The new shell.ts:865 string includes `"focus the footer Background
   tasks pill, then Enter — detail view + live updates"`. The em-dash
   is fine for English readers but non-ASCII chars in LLM-facing
   strings occasionally trip token-counting code paths and translation
   pipelines. Non-blocking; the existing codebase already uses em-dashes.
3. The `monitorRegistry.ts:197` rewording inverts the parenthetical
   from `"not visible from /tasks dialog reopens"` to `"not visible in
   later Background tasks dialog reopens or `/tasks` listings"`. The
   new claim is stronger: it asserts the chat-history notification is
   *also* not visible from `/tasks` text dumps. Verify that's actually
   true — if the text dump *does* include the notification, this is a
   regression in doc-truth. Worth a 30s sanity check before merge.
4. No tests touched. Per the description, "no test asserted on the
   exact string content of these LLM messages" — that's correct
   discipline (asserting on prose-y LLM strings is brittle). Approve
   as-is on test scope.

## Verdict

merge-after-nits
