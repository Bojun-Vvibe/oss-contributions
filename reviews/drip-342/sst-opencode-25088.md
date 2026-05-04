# sst/opencode #25088 — feat: editable allow all

- **Head SHA:** `f460217a38b663a8e650b819df317eec4adbd1b4`
- **Size:** +200 / -30 across 8 files
- **Verdict:** **needs-discussion**

## Summary
Replaces the read-only "Always allow" confirmation prompt in the TUI permission
dialog with an editable textarea (`AlwaysEditPrompt` in
`packages/opencode/src/cli/cmd/tui/routes/session/permission.tsx:442`). Users
can now hand-edit the pattern list before approving. The new optional
`patterns: string[]` field is plumbed through the API schema
(`permission/index.ts:63`, `httpapi/groups/permission.ts:14`,
`httpapi/handlers/permission.ts:23`) and a new `next.test.ts` exercises the
"override always patterns" path. SDK type files are regenerated.

## Strengths
- Schema change is backward-compatible: `patterns` is `Schema.optional`
  everywhere, and the server falls back to `existing.info.always` when the
  client omits it (`permission/index.ts:252-255`). Old clients keep working.
- The TUI handles the empty-input case sensibly — if the user clears the
  textarea, `patterns: undefined` is sent and the original `always` list is
  used (`permission.tsx:170-175`). Matches the previous behavior, so a user who
  accidentally hits Enter on empty input does not silently get a no-op
  approval.
- Newline handling is right: Alt+Enter inserts a literal newline, plain Enter
  confirms (`permission.tsx:480-489`). Documented inline at line 489 ("One
  pattern per line. Alt+Enter inserts a newline.") which is the kind of inline
  affordance text TUIs almost never have.

## Concerns (the "discussion" part)
- The textarea accepts arbitrary text. There is **no validation** that what
  the user typed parses as a real glob/permission pattern before it is sent to
  the server. A typo like `*.t s` or stray whitespace will silently install a
  pattern that never matches, and the user will be re-prompted on the next
  invocation with no obvious clue why. At minimum the trim+filter at
  `permission.tsx:172-175` should also reject patterns that fail
  `Permission.parsePattern` (or whatever the canonical validator is) and
  surface the failure inline.
- This is a **security-relevant UX**: the prior prompt offered a fixed
  server-vetted set of patterns; this one lets the user broaden them. A user
  who accepts the default would be unchanged, but a user who edits "*" into
  the box has now blanket-allowed the permission for the session. Worth a
  warning row when the edited list is strictly broader than the original
  (`bash` → `bash *` for example), or at minimum a one-time confirmation when
  `*` is added.
- The diff plumbs `patterns` through three layers but I do not see the
  server-side asserting that the user-supplied patterns are a subset of (or
  even related to) the original `existing.info.always`. A misbehaving
  client/plugin could send arbitrary patterns and get them approved. Is that
  intended? If yes, document it; if no, gate the override.
- `permission.tsx:457` declares `let input: TextareaRenderable` but only
  assigns it inside the `ref` callback (line 502). If the user hits Enter
  before the ref fires (effectively impossible in practice but easy to
  static-analyze), `input.plainText` will throw. A `?.` guard or null check
  would be cheap.

## Recommendation
Hold for a design pass on the validation/widening question above. The plumbing
and TUI mechanics are clean; the open question is the trust model for the
edited pattern list and whether the server should bound it.
