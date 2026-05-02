# sst/opencode PR #25493 — feat(plugin): pre_chat.messages.transform hook

- Head SHA: `e8799b21c52bdeecf0d2901f330bbbd89c15fc0e`
- URL: https://github.com/sst/opencode/pull/25493
- Size: +33 / -0, 1 file (`packages/plugin/src/index.ts`)
- Verdict: **request-changes**

## What changes

Adds a new plugin hook `pre_chat.messages.transform` (index.ts:281-309)
that receives the full materialized messages array (with parts) before
the LLM is invoked, so plugins can strip/replace `FilePart` objects
(image stripping, vision-description injection, system prompt rewrite).
The existing `experimental.chat.messages.transform` hook is marked
`@deprecated` (index.ts:310-313) but kept for backward compatibility.

The hook signature mirrors the input/output mutation pattern already
used elsewhere in the file: input carries `sessionID`, `agent`, `model`,
and `messages`; output exposes a writable `messages` array.

## What's good

- Solves a real gap. The existing `experimental.chat.messages.transform`
  hook (index.ts:310) was pretty much unusable because input is `{}` —
  plugins had no way to see what they were transforming. The deprecation
  comment ("has empty input and does not receive messages") is honest
  about why the old hook needs replacing.
- The `Promise<void>` return + mutation-via-output pattern is consistent
  with the surrounding hooks, so plugin authors won't be surprised.

## Blockers

1. **No implementation, only the type.** This PR only changes
   `packages/plugin/src/index.ts` — there is no corresponding wiring
   in `packages/opencode/src/plugin/...` (or wherever plugins are
   dispatched) that actually fires `pre_chat.messages.transform` before
   the LLM call. As written, plugins can declare the hook but it will
   never be invoked. Either the wiring is missing from the PR or it
   landed in a separate one — the PR description should make that
   explicit and the diff should include the dispatcher change.
2. The doc on lines 287-290 says "Remove `FilePart` objects with
   `image: true` to strip images. Replace with `TextPart` containing
   the vision description." but the type just says `Part[]`. Without
   a runtime example or test asserting that mutating
   `output.messages[i].parts` is honored downstream, plugin authors
   are guessing about whether they should *return a new array* or
   *mutate in place*. The JSDoc says "Return a new array or mutate"
   — pick one and document the precedence (the dispatcher needs to
   prefer one over the other).
3. The comment "Runs BEFORE `experimental.chat.messages.transform`"
   (line 291) defines an ordering relationship between the two hooks,
   but that ordering only matters if the dispatcher actually fires
   *both* in deprecation overlap. Confirm in code, not in JSDoc.

## Nits

- `Model` is referenced in the type without an import in the diff
  context — verify the file already imports it (it likely does, since
  it's used in adjacent hooks, but worth checking the final patch).
- Once wired, add a test in the plugin test suite asserting that an
  `image: true` FilePart is removed and the LLM payload reflects the
  transformation.

## Risk

Low (right now it's a type-only change), but the lack of a corresponding
runtime change makes this a footgun: plugin authors will adopt the new
hook, see nothing happen, and file bug reports. Either land the
dispatcher change in the same PR or hold this one until that change is
ready.
