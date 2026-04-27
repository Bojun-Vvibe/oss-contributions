# google-gemini/gemini-cli #25957 — feat(cli): implement event-driven hook system messages

- **Repo**: google-gemini/gemini-cli
- **PR**: #25957
- **Author**: achernez
- **Head SHA**: 893c0a867aea33383248413b305638651a9b4b51
- **Size**: +47 / −25 across four files.

## What it changes

Refactor moving the "hook fired a system message →
display it in chat history" plumbing from a return-value
threading pattern (where every caller of
`fireSessionStartEvent` had to remember to pull
`result.systemMessage` and call `historyManager.addItem`)
to a single subscriber on the `CoreEvent.HookSystemMessage`
event bus.

Three coordinated edits:

1. **`packages/core/src/utils/events.ts:355`**: switch
   `emitHookSystemMessage` from a synchronous
   `this.emit(...)` to `this._emitOrQueue(...)`. The
   `_emitOrQueue` helper queues events emitted before any
   subscriber has registered (i.e. during early bootstrap
   before `AppContainer` mounts) and replays them when the
   first subscriber attaches. Without this, hooks that
   fire during initial `fireSessionStartEvent` would emit
   into the void and the system message would silently
   drop.

2. **`packages/cli/src/ui/AppContainer.tsx:498-507`**:
   removes the inline `if (result.systemMessage) { ... }`
   branch that previously ran inside the
   `fireSessionStartEvent` resolution. The `AppContainer`
   already subscribes to `CoreEvent.HookSystemMessage`
   elsewhere (the test asserts `mockCoreEvents.on.mock.calls.find(c => c[0] === CoreEvent.HookSystemMessage)`
   succeeds), so the inline path was double-emitting:
   once via the event bus, once via the return value.

3. **`packages/cli/src/ui/commands/clearCommand.ts:55-75`**:
   same removal — `await hookSystem.fireSessionStartEvent(...)`
   no longer captures `result`, and the trailing
   `if (result?.systemMessage) { context.ui.addItem(...) }`
   block is deleted. The `MessageType` import becomes
   dead and is dropped.

## Strengths

- **Right architectural direction.** The pre-fix shape
  ("hook fires → returns systemMessage → caller adds to
  history") was a return-value side-channel that every
  call site of `fireSessionStartEvent` had to honor
  identically — and the fact that the deleted blocks in
  `AppContainer.tsx` and `clearCommand.ts` were
  byte-for-byte structurally identical (`if
  (result.systemMessage) { addItem({type: INFO, text: ...}) }`)
  is direct evidence that this was a duplication
  begging to be lifted into a single subscriber.
- **Bootstrap-ordering fix is non-trivial.** The
  `this.emit → this._emitOrQueue` change at
  `events.ts:355` is the load-bearing edit that makes
  the refactor *safe*: without it, hooks fired during
  the very-first `fireSessionStartEvent` call (before
  `AppContainer`'s `useEffect` has run to attach the
  listener) would silently disappear under the new
  shape. The PR author identified this and switched to
  the queueing emit variant, which suggests careful
  thought about init ordering.
- **Test pins the new contract end-to-end** at
  `AppContainer.test.tsx:3580-3622`: locates the
  registered `CoreEvent.HookSystemMessage` handler via
  `onSpy.mock.calls.find(call => call[0] === CoreEvent.HookSystemMessage)?.[1]`,
  invokes it with a synthetic payload
  `{hookName: 'test-hook', eventName: 'SessionStart', message: 'Test hook message'}`,
  and asserts `mockAddItem` is called with
  `{type: MessageType.INFO, text: 'Test hook message', source: 'test-hook'}`.
  The `source: 'test-hook'` field is interesting — it
  threads the originating hook name into the history
  item, which the previous return-value path didn't
  carry. This is a small UX win (users can see *which*
  hook produced a system message) and the test pins it.

## Concerns / nits

- **No regression test for the bootstrap-ordering case.**
  The `_emitOrQueue` switch is correct but there's no
  pin-test that "an event emitted before any subscriber
  attaches is replayed when the first subscriber
  attaches". A single test that calls
  `coreEvents.emitHookSystemMessage(payload)` *before*
  rendering `AppContainer`, then renders, then asserts
  `mockAddItem` was eventually called, would lock the
  load-bearing piece of the refactor against a future
  "simplification" PR that flips `_emitOrQueue` back
  to `emit`.
- **The deleted `clearCommand.ts` path** had a
  `// Give the event loop a chance to process any
  pending telemetry operations` comment immediately
  before the now-deleted addItem. The comment stays,
  but its reason (the addItem ran after the event-loop
  yield to make sure telemetry flushed first) no longer
  applies because the addItem is now driven by the
  event bus and runs whenever the subscriber decides.
  Worth either deleting the now-stale comment or
  rewording it to explain what *currently* needs the
  yield (presumably the `uiTelemetryService.clear`
  on the next line).
- **Subscriber lifecycle on unmount.** The test
  `unmount()`s the container but doesn't assert that
  the `HookSystemMessage` subscriber was *removed*. If
  `AppContainer` mounts and unmounts repeatedly (e.g.
  during a TUI rerender storm or a hot-reload dev
  loop), each mount adds a new subscriber and any stale
  ones keep firing into stale `historyManager`s. A
  `expect(coreEvents.off).toHaveBeenCalledWith(CoreEvent.HookSystemMessage, hookSystemMessageHandler)`
  assertion in the unmount path would close that gap.
- **Naming.** The new `source` field on the history item
  (`source: 'test-hook'`) doesn't appear in any other
  diff hunk — this PR introduces it implicitly via the
  test. If `MessageType.INFO` items rendered elsewhere
  in the TUI don't read `source`, the field is dead
  data. A grep across `packages/cli/src/ui/messages/`
  for `.source` would confirm there's a renderer.

## Verdict

**merge-after-nits.** Right refactor, correct
load-bearing init-ordering fix via `_emitOrQueue`, and
a clean removal of duplicated history-add code at two
sites. Wants a bootstrap-ordering pin test, an
unmount-unsubscribe assertion, and confirmation that
the new `source` field has a renderer reading it
before merge.

## What I learned

When a side-channel pattern (return-value `systemMessage`)
ends up duplicated across N callers because every caller
has to pattern-match the same shape, the refactor to a
shared subscriber on a typed event bus is almost always
worth it — but only if the bus has a queue-before-subscribe
shape (here `_emitOrQueue`). Naive pub/sub on early
bootstrap events silently drops messages emitted before
the first subscriber attaches, and the symptom is "the
first session-start hook never shows up but subsequent
ones do" — a flaky-looking bug that's actually
deterministic ordering. The `_emitOrQueue` discipline is
the cheap fix; lift it to a project convention so future
event-bus migrations don't re-discover the same trap.
