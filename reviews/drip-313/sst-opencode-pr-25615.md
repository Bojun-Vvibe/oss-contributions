# sst/opencode PR #25615 — feat: add /btw side question command

- Author: NihalMohan04
- Head SHA: `dcb13d4a8ae673ae27657ef77bf39033ff264610`
- Verdict: **merge-after-nits**

## Rationale

Adds a `/btw` slash command + `DialogBtw` TUI dialog that asks a one-shot
"side question" against the current session without polluting history. New
file `packages/opencode/src/cli/cmd/tui/component/dialog-btw.tsx:1-66` is
clean Solid + opentui, dismissed on enter/escape/space at lines 16-22. The
prompt-router hook in `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:836-849`
matches `/btw <q>` correctly but `inputText.slice(5).trim()` on the bare
`/btw` token (no trailing space) yields an empty string and silently no-ops
— consider showing the prompt dialog in that case the same way the command-
palette path does at `routes/session/index.tsx:528-543`. Server-side
registration in `command/index.ts:103-110` and `httpapi/groups/session.ts:65`
wires the schema cleanly; the `$ARGUMENTS` template is sensible. Ship after
the empty-arg UX nit is addressed.
