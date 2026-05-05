# google-gemini/gemini-cli #26480 — feat(core): steer model to surgical edits and prevent accidental deletions

- **Repo**: google-gemini/gemini-cli
- **PR**: [#26480](https://github.com/google-gemini/gemini-cli/pull/26480)
- **Author**: aishaneeshah
- **Head SHA**: `6a51bcb5fa0c5103225f5b8bc92d05237fb0febc`
- **Base branch**: `main`
- **Created**: 2026-05-05T02:20:00Z (approx; in same window as listing)

## Verdict

`merge-after-nits`

## Summary of change

Pure prompt/tool-description tuning for the `gemini-3` model family. Two
deltas:

1. `packages/core/src/prompts/snippets.ts:235` — the read-pattern guideline
   `"read_file fails if old_string is ambiguous, causing extra turns"` is
   corrected to `"replace fails if old_string is ambiguous, ..."`. This is a
   straightforward bug fix in the steering text — `old_string` is a parameter
   of `replace`, not `read_file`, so the previous sentence pointed the model
   at the wrong tool to be careful with.
2. `packages/core/src/tools/definitions/model-family-sets/gemini-3.ts:123` and
   `:358` — `write_file` and `replace` descriptions are extended:
   - `write_file`: appends "to minimize token usage and simplify reviews" to
     the existing recommendation to use `replace` for targeted edits to large
     files.
   - `replace`: inserts "This tool is preferred for surgical edits to existing
     files as it minimizes token usage, simplifies code reviews, and avoids
     accidental deletions." after the `allow_multiple` sentence.

Snapshot regeneration consumes the rest of the diff (15 of 18 hunks in
`packages/core/src/core/__snapshots__/prompts.test.ts.snap`, plus the two
hunks in `packages/core/src/tools/definitions/__snapshots__/coreToolsModelSnapshots.test.ts.snap`),
all of which are mechanically derived from the source change.

## What's good

- The `read_file` → `replace` correction in `snippets.ts:235` is a real
  factual fix; the previous sentence was nonsense in context.
- The new "avoids accidental deletions" framing addresses a known failure
  mode where the model rewrites a whole file via `write_file` and silently
  drops sections it didn't reproduce verbatim. Steering toward `replace`
  reduces the blast radius.
- Scoped to the `gemini-3` model family set only, so other model families'
  prompts are unaffected. Low-risk.
- Snapshot regeneration is complete (16 snap blocks updated), so CI snapshot
  tests will pass.

## Nits / questions

- The new `replace` description ("preferred for surgical edits to existing
  files") is positioned between the `allow_multiple` sentence and the
  "requires providing significant context" sentence. The original sentence
  ordering was: (1) what it does, (2) `allow_multiple`, (3) context
  requirement. The insertion now reads (1) what it does, (2) `allow_multiple`,
  (3) preferred for surgical edits, (4) context requirement. Consider
  reordering to put the "preferred for ..." sentence right after sentence (1)
  — that's the model's first-pass framing of when to use the tool, which
  is the highest-value steering position.
- `write_file`'s tail "use 'replace' for targeted edits to large files to
  minimize token usage and simplify reviews" repeats the same justification
  the `replace` description uses. The model will see both descriptions when
  ranking tools; consider keeping `write_file`'s description short
  ("Best for new or small files; for targeted edits to existing files,
  prefer `replace`.") and putting the rationale only in `replace`.
- No A/B telemetry or eval is referenced in the PR body, so we're trusting
  the qualitative argument that this reduces accidental deletions. A small
  eval against a "rewrite this large file with a one-line change" benchmark
  would harden the case for future steering changes; not blocking.
- The fix on `snippets.ts:235` should arguably be a separate commit / PR
  from the `replace` description steering — they're different changes (one
  is a bug fix to a generic snippet, the other is family-specific steering)
  with different review surfaces. Not blocking.

## Risk

Very low. Prompt-only change, scoped to one model family, snapshots updated
in lockstep with the source.

## Recommendation

Land. Optionally consider the sentence-ordering nit and splitting the
generic-snippet bug fix into its own PR for cleaner history.
