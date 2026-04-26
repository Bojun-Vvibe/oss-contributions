# block/goose #8846 — fix(ui): keep SSE reconnect loop alive on long disconnects (#8717)

- **Repo**: block/goose
- **PR**: #8846
- **Author**: fresh3nough
- **Head SHA**: 37309e41176506560e89eb0cabf4859627ddaa25
- **Base**: main
- **Size**: +196 / −63 — the production change is mostly *deletions*
  in `ui/desktop/src/hooks/useSessionEvents.ts` (–~50 lines), plus a
  new test file `useSessionEvents.test.tsx` (+167) covering the
  regression behavior.

## What it changes

Removes the `MAX_CONSECUTIVE_ERRORS = 10` reconnect cap and the
two synthetic-`Error`-event broadcast blocks in
`ui/desktop/src/hooks/useSessionEvents.ts`. Both the "stream ended
with no events" path (previously `:103-122`) and the catch-block
path (previously `:127-147`) used to fan out a synthetic
`{ type: 'Error', error: 'Lost connection to server' }` to every
registered listener once 10 consecutive failures hit. The new code
just backs off and reconnects forever, with a top-of-loop comment at
`:30-35` explaining why fatal errors are *only* signalled by real
`Error` events from goosed (replay-buffer overflow).

The companion test file
(`ui/desktop/src/hooks/useSessionEvents.test.tsx`) provides two
regression tests:

- 25 silent-failure reconnects past the old threshold; asserts the
  registered listener never receives an `Error`-typed event
  (`useSessionEvents.test.tsx:111-128`).
- 12 silent failures followed by recovery yielding a real `Message`
  event; asserts the listener gets the `Message` and never gets a
  synthetic `Error` (`:147-167`).

## Strengths

- The fix is structurally correct: the bug (#8717) was that the
  hook's *transport-layer* failure was being escalated to a
  *protocol-layer* terminal error, which then propagated through
  `useChatStream` and killed the in-flight chat. Removing the
  escalation is exactly the right move because SSE-with-`Last-Event-ID`
  *is* the recovery protocol — the loop already does the right
  thing if you just let it keep trying.
- The test infrastructure is well-thought-out:
  - `boundedMock` (`useSessionEvents.test.tsx:43-58`) wedges the
    loop on a never-resolving promise after N attempts so the test
    has a deterministic stop point even with patched `setTimeout`.
  - `setTimeout` is monkeypatched to `queueMicrotask`
    (`:78-82`) so backoff doesn't make the test slow but still
    exercises the await chain.
  - `flush()` (`:65-69`) drains microtasks rather than using
    `vi.useFakeTimers()` — simpler and avoids the well-known
    fake-timers-vs-async-iterator interaction bugs.
- The docstring at the top of the test (`:75-80`) explicitly cites
  issue #8717 and explains *why* the synthetic-error broadcast was
  pathological. Future maintainers tempted to reintroduce a
  reconnect cap will see the regression test and the rationale.
- `lastEventId` plumbing on the `Last-Event-ID` header
  (`:42-44`) is preserved — that's the part that makes infinite
  reconnect *correct*, not just *non-fatal*: the server can replay
  the events the client missed during the disconnect window.

## Concerns / asks

- **Connection state is no longer surfaced via the listener
  channel.** The previous (buggy) behavior at least gave the UI a
  signal "connection is in trouble" via the synthetic Error. With
  this fix, a 30-minute silent disconnect just looks like "no new
  events" to the listener; only the `setConnected(false)` /
  `setConnected(true)` boolean (the hook's other return value) tells
  the UI anything is wrong. Worth confirming in the PR description
  (or adding to the docstring) that callers who care about the
  banner UX *do* observe `connected` and aren't relying on the old
  Error events. A grep for consumers of `useSessionEvents` and a
  one-line note in the diff would close this.
- The "stream ended without events" branch at `:107-110` now does
  exactly the same thing as the catch branch — backoff and retry.
  Worth folding the two into a single helper or, at minimum, making
  the comment block at the top reference both code paths so a
  future reader sees one mental model, not two.
- The catch branch at `:115-119` no longer logs the
  `consecutiveErrors/MAX` ratio (since the cap is gone), but it
  also drops the count itself. For an operator triaging "why is my
  chat dropping events", knowing "we've been retrying for 47 cycles"
  is more useful than "we got an SSE error" repeated 47 times.
  Consider keeping a local counter just for the log (no behavioral
  use), e.g. `console.warn('SSE reconnect attempt N, reconnecting:', error)`.
- Backoff caps at `MAX_RETRY_DELAY = 10_000` (10s). For a
  multi-hour disconnect (laptop closed overnight), that's 360
  reconnect attempts per hour, every hour, until the user wakes the
  machine. Probably fine because each attempt is a single fetch and
  the hook is gated on visibility/focus elsewhere — but worth
  confirming in the PR description that `useSessionEvents` either
  doesn't run during App Nap, or that the rate is acceptable. A
  `document.visibilityState`-gated long-sleep path
  (e.g. delay → 60s when hidden) would be an easy follow-up.
- The test patches global `setTimeout`. If another module is
  imported into the test that schedules its own timers, this will
  affect them too. Restore in `afterEach` is correct
  (`:84-86`); worth a comment that the patch is process-wide and
  not lib-local.

## Verdict

**merge-as-is** — the fix is small, surgical, and correctly
targets the actual bug (synthetic terminal error masquerading as a
protocol error). The tests cover the regression directly. Asks are
follow-ups, not blockers — the connected-flag surface area exists
for the UX signal the synthetic Error was wrongly providing, and
the visibility-gated backoff is a separate optimization.

## What I learned

A reconnect cap on a transport that *has* a replay protocol
(SSE + `Last-Event-ID`) is almost always wrong: the cap conflates
"transport is having a bad time" with "protocol is dead". The right
escalation point is the protocol layer signalling something fatal
(here, server-side replay buffer overflow → real `Error` event
from goosed) — the transport layer should just keep retrying. The
related anti-pattern ("count failures, escalate at threshold") is
common and almost always papers over the fact that the *consumer*
hasn't been given a separate "still trying" channel.
