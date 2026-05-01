# sst/opencode #25198 — fix: fix AI refusing to commit

- **Repo:** sst/opencode
- **PR:** https://github.com/sst/opencode/pull/25198
- **HEAD SHA:** `dbf6fc674349b1c7e70e8d4862dd1a55631bf188`
- **Author:** scarf005
- **Verdict:** `merge-after-nits`

## What the diff does

Three-file prompt-policy edit removing the absolute "NEVER commit changes
unless the user explicitly asks you to" line from two system prompts plus
relaxing the same line out of `bash.txt`'s git-commit how-to:

1. `packages/opencode/src/session/prompt/default.txt:86` — deletes the
   "NEVER commit changes…" sentence (and the trailing blank line) from the
   default system prompt.
2. `packages/opencode/src/session/prompt/trinity.txt:78` — same deletion from
   the trinity prompt.
3. `packages/opencode/src/tool/bash.txt:53/66` — softens the lead from
   `Only create commits when requested by the user. If unclear, ask first.`
   to `When creating a new git commit, follow these steps carefully:` and
   removes the redundant Git-Safety-Protocol bullet that mirrored the same
   "NEVER commit unless asked" assertion.

Linked issue is #17157 (AGENTS.md says "always commit after a task" but the
agent refuses because the system prompt's NEVER beats the user's project
guide).

## Why the change is right

The root cause is a **policy-resolution-order** drift, not a wording bug:
the system prompt's "NEVER commit unless explicitly asked" is an absolute,
while AGENTS.md is project-scoped guidance. As written, the absolute always
wins and project-level "always commit on completion" intent gets silently
overridden — which is the exact user complaint. Deleting the absolute and
keeping only the procedural Git-Safety-Protocol bullets at `bash.txt:55-65`
(which are about *how* to commit safely, not *whether* to) restores the
"project AGENTS.md is the source-of-truth for commit cadence" contract.

The lead-line softening at `bash.txt:53` is the load-bearing part: the prior
`Only create commits when requested by the user. If unclear, ask first.` was
a *prescription* (always ask), the new `When creating a new git commit,
follow these steps carefully:` is a *contract* (when you commit, do it like
this). The contract form lets project AGENTS.md override the cadence
without contradicting the safety procedure.

## Nits (non-blocking)

1. **Asymmetric removal across prompts.** Three prompt files exist under
   `packages/opencode/src/session/prompt/`; this PR edits two
   (`default.txt`, `trinity.txt`). A quick `grep -l "NEVER commit changes"
   packages/opencode/src/session/prompt/` would confirm no third prompt
   still carries the deleted line and silently re-introduces the override
   bug for users on that prompt variant.

2. **No replacement guidance at the prompt level.** Without the absolute,
   the prompt now leaves commit cadence entirely to AGENTS.md. A one-line
   replacement like *"Defer to AGENTS.md for project-specific commit
   cadence; if AGENTS.md is silent, default to asking before committing."*
   at `default.txt:86` would prevent the opposite drift (over-eager commits
   on projects with no AGENTS.md).

3. **`bash.txt:66` deletion is the same sentence twice.** The deleted bullet
   "NEVER commit changes unless the user explicitly asks you to. It is VERY
   IMPORTANT…" was already a duplicate of the system-prompt assertion. PR
   description should name this so the reviewer doesn't wonder whether the
   safety semantics actually changed (they didn't — the procedural bullets
   `:54-64` all stay).

## Verdict rationale

Surgical three-file fix targeting a real and reproduced policy-resolution
drift (#17157). Worth a once-over for prompt-symmetry across all variants
and an optional inline pointer to AGENTS.md, but the change itself is the
right shape and restores the documented project-config-wins precedence.

`merge-after-nits`
