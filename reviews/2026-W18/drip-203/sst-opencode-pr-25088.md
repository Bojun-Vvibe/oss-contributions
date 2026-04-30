# sst/opencode #25088 — feat: editable allow always

- **Author:** al-prk
- **SHA:** `71ec904`
- **State:** OPEN
- **Size:** +200 / -30 across 8 files (TUI prompt, permission service, HttpApi
  group + handler, REST route, behavioral test, generated SDK)
- **Verdict:** `merge-after-nits`

## Summary

Closes #25089. Replaces the read-only "confirm server-suggested patterns"
`AlwaysPrompt` with a new `AlwaysEditPrompt` textarea component
(`permission.tsx:442-535`) that lets the user edit the `props.request.always`
list before confirming, then threads an optional `patterns?: readonly string[]`
field through the entire reply chain — `Permission.ReplyBody` schema at
`src/permission/index.ts:63`, the `Permission.layer.reply` effect at
`:252-256`, the HttpApi `ReplyPayload` Struct at
`server/routes/instance/httpapi/groups/permission.ts:14`, the handler payload
forward at `handlers/permission.ts:23`, the REST route at
`server/routes/instance/permission.ts`, and the regenerated SDK
`PermissionReplyData.body.patterns` at `sdk/js/src/v2/gen/types.gen.ts:4293` +
`Permission.reply()` at `sdk.gen.ts:2725,2739`. Locked behaviorally by two new
`it.live` cases in `test/permission/next.test.ts:1125-1206` covering both
override (`patterns: ["docker exec -it app_* cat *"]` rather than the
server-suggested `"docker *"`) and empty-array fallback (`patterns: []` falls
back to the server-suggested set).

## Reasoning

The shape is correct: optional field, server-side fallback to existing
`existing.info.always` when `input.patterns` is unset *or* empty, and
generated SDK regenerated cleanly with both `body` field declaration and
`bodySerializer` registration kept in lockstep. The two regression tests
exercise the load-bearing semantics in both directions and use the established
`waitForPending(1)` + `Fiber.join` pattern from the rest of `next.test.ts`.

Three nits worth a follow-up before merge:

1. **`AlwaysEditPrompt`'s confirm path at `permission.tsx:486-491` strips
   blank lines but does no further validation** — a user who pastes a long
   line wrapped to two with a soft return will silently drop the second half.
   Cheap fix: validate that each kept pattern starts with the same command
   prefix as `props.permission`, or just show a "{n} pattern(s) will be
   allowed" preview line in the dialog body so the user can sanity-check
   before pressing enter.

2. **Empty-array semantics could use a one-line comment at
   `index.ts:252-254`.** The fallback rule is "empty array means use
   server-suggested" — that's reasonable but it's also indistinguishable from
   "user explicitly cleared all patterns". Today `patterns: []` and
   `patterns: undefined` both fall through to `existing.info.always`. The
   second test locks this behavior, so it's intentional, but a future caller
   reading the schema will assume `patterns: []` means "allow nothing this
   time" — the comment should name the choice ("empty array is treated as
   absent; pass `reply: 'reject'` to deny").

3. **The new `patterns` field is not surfaced in `audit_log` shape** — the
   reply event recorded for the session permission still references
   `existing.info.always` rather than `patternsToAllow`, so a permission
   audit trail will not see what the user actually approved. This is
   pre-existing audit-log scope rather than this PR's invention but it's
   worth filing a follow-up because the value of "allow-always editable" is
   diluted by an audit trail that doesn't record the edit.

The TUI textarea component itself matches the existing `RejectPrompt` shape
exactly (same `useKeyboard` escape/return handling, same `dialog.stack.length`
guard, same narrow/wide responsive layout at `dimensions().width < 80`), so
the surface keeps a consistent feel. Net: ship after the validation/comment
nits, audit-log gap is a separate ticket.
