# sst/opencode#24711 — fix(tui): keep Zed context polling responsive

- **Repo**: sst/opencode
- **PR**: [#24711](https://github.com/sst/opencode/pull/24711)
- **Head SHA**: `0e5c4f71911d9a0443c75444165cb16b62346892`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-as-is`

## Context

`packages/opencode/src/cli/cmd/tui/context/editor.ts` connects to the
Zed editor over a websocket to mirror cursor position, current file,
and selection state into the TUI's editor context. When the websocket
closes (Zed restart, reconnect, transient drop) the connect loop runs
through `scheduleReconnect`, an exponential-backoff helper.

The poll path — the periodic state-DB read used when no websocket is
available — was *also* using `scheduleReconnect`. Same backoff
behavior, same `delay` variable being doubled on each iteration. So
the longer Zed stayed unresponsive on the websocket, the further apart
the state-DB polls drifted. After a few cycles the context could be
several seconds stale, which is the user-visible bug.

## The fix

Add a dedicated `scheduleZedPoll` helper that fires at a fixed 1s
cadence, decoupled from reconnect backoff:

```ts
const scheduleZedPoll = () => {
  if (closed) return
  if (reconnect) clearTimeout(reconnect)
  reconnect = setTimeout(connect, 1000)
}
```

…and swap the call site in the no-websocket branch from
`scheduleReconnect()` to `scheduleZedPoll()`. The shared `reconnect`
timer handle is reused so the close/cancel path still works without
adding a second handle.

## What this means at runtime

| Branch | Old delay | New delay |
|---|---|---|
| websocket reconnect after close | 100ms → 200 → 400 → … (exp backoff) | unchanged |
| Zed state-DB poll, no websocket | same exp backoff | fixed 1000ms |

So the cursor/file mirror updates every second instead of degrading
into multi-second windows. Reconnect attempts to the actual websocket
endpoint still back off, which is the right semantics — backoff is
for "the remote isn't here", not for "we're polling a local DB
because the remote isn't socket-served".

## Risk analysis

- **Timer handle reuse**: `scheduleZedPoll` and `scheduleReconnect`
  both write to the same `reconnect` variable. Any caller that
  schedules then re-schedules will correctly cancel the prior
  timeout via the `clearTimeout` in each helper. The shared handle
  also means the existing close-cleanup code (`clearTimeout(reconnect)`
  on disconnect/teardown) still cancels regardless of which helper
  scheduled the active timer. Good.
- **Variable name**: keeping the variable named `reconnect` even
  though it now sometimes holds a poll timer is a minor smell. Worth
  renaming to something neutral like `pendingTimer` in a follow-up,
  but not blocking this PR.
- **CPU/load**: 1Hz state-DB poll is fine for the use case (cursor
  position changes are interactive-rate, sub-second user perception
  threshold). If Zed grows more state to mirror, this becomes worth
  re-tuning.
- **Closed-flag check**: both helpers correctly bail on `closed` so
  there's no zombie poll after the editor context tears down.

## Suggestions (non-blocking)

- The `1000` literal should probably be a named constant
  (`ZED_POLL_INTERVAL_MS = 1000`) for the same reason the layout
  breakpoints in any other TUI surface should be — it's a tunable
  user-facing latency, and grepping for it later is easier when it
  has a name.
- The PR's test claim is `bun run test test/cli/tui/editor-context.test.ts`
  + `bun typecheck`. I'd want the test to assert that the poll fires
  on a fixed cadence regardless of how many prior reconnect attempts
  have occurred — i.e. drive the timer through fake-time and assert
  the `connect` function gets called every 1000ms after the
  no-websocket branch lands. Not blocking but it would lock the
  fix in.

## Verdict

`merge-as-is`. Minimal, targeted fix to a real responsiveness bug.
Reuses the existing timer-handle plumbing correctly, doesn't change
the reconnect-backoff semantics, and the closed-flag guard is in
place. The `1000` literal and the `reconnect` variable name are
both worth a tiny follow-up but neither is worth blocking.

## What I learned

When a single helper is overloaded for two semantically different
schedules ("retry an unreachable resource" vs "poll a reachable
resource at a fixed cadence"), the shared backoff state is the
gotcha — the second use case ends up paying for the first's failure
mode. The fix is almost always "two helpers, one timer handle"
rather than "one helper with a flag". Worth grepping any poll/retry
loop for callers that share a backoff variable.
