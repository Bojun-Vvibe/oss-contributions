# sst/opencode#25128 — fix(acp): include tool image attachments in updates

- PR: https://github.com/sst/opencode/pull/25128
- Head SHA: `4c5b629ac8c5dd72944809f0c662e46bcbb672cc`
- Author: SteffenDE
- Files: `acp/agent.ts` +68/−62, `acp/event-subscription.test.ts` +178/−1; +246/−63
- Closes #25127

## Context

The ACP agent was emitting completed-tool updates with a hand-rolled
`ToolCallContent[]` builder duplicated at `agent.ts:344` and `agent.ts:873`
(the two `case "completed":` arms of the part-update reducer). Both arms
shipped `{type:"text", text: part.state.output}` — so any image bytes returned
by a tool (e.g. screenshot tools used by Tidewave) were stringified into the
text envelope and lost on the ACP wire. Same site also called
`{output: part.state.output, metadata: part.state.metadata}` literally for
`rawOutput`, again duplicated.

## Design

The diff factors both duplicated blocks into two helpers at
`agent.ts:1533-1602`:

- `completedToolContent(part, kind)` returns the `ToolCallContent[]`. Early-
  exits with `[]` if `part.state.status !== "completed"` (defensive — both
  callers are already inside `case "completed":`, but the guard means the
  helper is reusable from any update site without re-checking). Builds the
  text envelope unconditionally, then for `kind === "edit"` appends the
  `{type:"diff", path, oldText, newText}` shape with the same
  `oldString | newString | content` triple-fallback the inline code had.
- `completedToolRawOutput(part)` returns `{output, metadata}` and — the
  load-bearing new branch — when `part.state.metadata?.image` is present,
  attaches an `ToolCallContent` of `{type:"image", data, mimeType}` to the
  content array. (Verified by the test additions at
  `event-subscription.test.ts:1-178` which assert the agent forwards the
  base64 + mime through to the ACP `session/update` notification.)

Both `case "completed":` arms now reduce to two lines:
`const content = completedToolContent(part, kind)` and
`rawOutput: completedToolRawOutput(part)`. The `todowrite` parsed-todos
branch and the catch-all error logger are untouched.

## Risks / nits

- The two completed-arms remain literally duplicated except for the helper
  calls (the `toolStarts.delete` / `bashSnapshots.delete` / `kind` / status
  dispatch is byte-identical at `agent.ts:344-413` and `agent.ts:843-911`).
  Worth a follow-up to factor the whole arm, not just the content builder —
  but out of scope for a bug-fix PR and the current factoring is the
  minimum-blast-radius change.
- `completedToolContent`'s early `return []` for non-completed status is
  unreachable from the two existing callers but defensive against future
  reuse. A one-line `// defensive: callers already gate on status` comment
  would prevent a future reader from "fixing" the dead branch.
- The image attachment shape is inferred from `part.state.metadata.image` —
  no schema validation that the payload is base64. If a tool returns a raw
  `Buffer` or a data URL (`data:image/png;base64,...`) the ACP frame will
  ship a malformed `data` field. A `typeof data === "string" && !data
  .startsWith("data:")` precondition with a fallback that strips the data-
  URL prefix would harden this without changing the happy path.

## Verdict

**`merge-after-nits`**

The bug is real (image bytes silently stringified into the text envelope on
ACP wire), the fix is the minimal de-duplication that makes a single
attachment point reachable, the test additions exercise the new shape end-
to-end against a faked event-subscription, and the helper extraction keeps
the two reducer arms in sync going forward (any future content-shape
change happens in one place). The image-payload shape-validation nit and
the wider duplicate-arm factoring are both follow-up scope.

## What I learned

When a code path has two literal duplicates and you need to add a third
behavior to it, factoring the *common shape* (here: the content array
builder + raw-output builder) into helpers and leaving the dispatch arms
as thin wrappers is a better refactor than trying to merge the arms
themselves — the arms have subtly different surrounding context
(`.catch((error) => ...)` vs `.catch((err) => ...)`, different log
prefixes) that would expand the diff and the test surface.
