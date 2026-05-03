# Review: sst/opencode #25628 — Add v2 session failure events

- PR: https://github.com/sst/opencode/pull/25628
- Head SHA: `2364c2cc9b89ac0d56bcd25f2b60746107b7bbe1`
- Author: thdxr (Dax)
- Size: +104 / -22

## Summary
Adds a v2 `session.next.step.failed` event and renames the existing
`session.next.tool.error` to `session.next.tool.failed` for naming
consistency. Both event schemas now share a hoisted `Error` struct
(`{ type, message }`). The TUI `sync-v2.tsx` projector consumes the new
step-failed event to set `currentAssistant.finish = "error"` and stamp
`time.completed`.

## Specific citations
- `packages/opencode/src/v2/session-event.ts:25-28` — extracts a shared
  `Error = Schema.Struct({ type, message })` and reuses it in both
  `Step.Failed` and `Tool.Failed`. Good dedupe.
- `packages/opencode/src/v2/session-event.ts:136-145` — new
  `Step.Failed` event, aggregate=`sessionID`, schema includes the shared
  `Error`. No `timestamp` field declared in the schema even though the
  emitter at `processor.ts:660` passes one — relying on the `Base` spread
  to carry it. Worth confirming `Base` actually has `timestamp` or this
  is silently dropped at validation.
- `packages/opencode/src/v2/session-event.ts:293-309` — `Tool.Error` →
  `Tool.Failed` rename; the type alias `export type Failed` is also
  renamed. Any external consumer of the old `SessionEvent.Tool.Error`
  type will break — this is a breaking change for the v2 event package.
- `packages/opencode/src/session/processor.ts:408` — dual-write switched
  to `SessionEvent.Tool.Failed.Sync`. Comment still says "Temporary
  dual-write while migrating session messages to v2 events" which is
  accurate.
- `packages/opencode/src/session/processor.ts:653-664` — emits
  `Step.Failed.Sync` only when `!ctx.assistantMessage.summary`. The
  guard is undocumented; intent appears to be "don't emit step.failed
  for summary turns" but a one-line comment would help reviewers.
- `packages/opencode/src/session/projectors-next.ts:164-166` — new
  projector for `Step.Failed.Sync` mirroring the existing pattern.
- `packages/opencode/src/cli/cmd/tui/context/sync-v2.tsx:146-154` —
  consumer sets `finish="error"` and copies `event.properties.error` to
  the assistant. This assumes `error` is structurally compatible with
  whatever shape `currentAssistant.error` previously held (the v1
  `Error` instance). If the consumer used `error.stack` anywhere, it
  will now be `undefined`.

## Verdict
**merge-after-nits**

## Nits
1. `Tool.Error` → `Tool.Failed` is a breaking rename in the public
   `SessionEvent` namespace; call it out in the PR body / changelog.
2. Add a one-line comment to `processor.ts:653` explaining why the
   `!ctx.assistantMessage.summary` guard exists.
3. Confirm `Base` carries `timestamp`; if not, declare it on the
   `Step.Failed` schema explicitly (the projector at
   `sync-v2.tsx:150` reads `event.properties.timestamp`).
