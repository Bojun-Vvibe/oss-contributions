# charmbracelet/crush #2663 — fix(app): replace single events channel with pubsub.Broker for fan-out

- **Repo**: charmbracelet/crush
- **PR**: [#2663](https://github.com/charmbracelet/crush/pull/2663)
- **Head SHA**: `0ea62dae930f980f68dd7b56ad68735a27e38d1b`
- **Author**: taigrr
- **State**: OPEN (+157 / -152)
- **Verdict**: `merge-as-is`

## Context

The SSE multi-consumer bug: `App.events` was a single
`chan tea.Msg` shared by every caller of `Events()` /
`SubscribeEvents()`. Multiple SSE clients on the same workspace
each got *some* fraction of events because Go channel reads are
competing-consumer (one receiver wins per send). The intended
shape is fan-out — every subscriber sees every event.

## Design

Replaces the channel with `*pubsub.Broker[tea.Msg]` and rewires
all senders/receivers.

1. **Field swap** at `internal/app/app.go:69`: `events chan tea.Msg`
   → `events *pubsub.Broker[tea.Msg]`. Constructor at line 103
   updates from `make(chan tea.Msg, 100)` to `pubsub.NewBroker[tea.
   Msg]()`.

2. **Public API change** at lines 156-161: `Events() <-chan tea.Msg`
   becomes `Events(ctx context.Context) <-chan pubsub.Event[tea.
   Msg]`. Each call now returns a *per-caller* channel via
   `app.events.Subscribe(ctx)`. The returned element type is now
   `pubsub.Event[tea.Msg]` (so callers see the wrapper, not the
   bare msg). This is a breaking signature change — anyone
   external calling `app.Events()` will fail to compile, which is
   the right outcome for a fan-out bug.

3. **`SendEvent`** (lines 162-167) loses the `select { case ch
   <- msg: default: }` non-blocking pattern in favor of
   `app.events.Publish(pubsub.UpdatedEvent, msg)`. The broker's
   per-subscriber buffer absorbs slow consumers without dropping
   on the publisher side.

4. **`setupSubscriber`** (lines 488-505) is dramatically
   simplified. The old code maintained a `subscriberSendTimeout =
   2 * time.Second` per-message timer and silently dropped
   messages when the consumer was slow (old line 522:
   `slog.Debug("Message dropped due to slow consumer", ...)`).
   The new version is just `broker.Publish(pubsub.UpdatedEvent,
   tea.Msg(event))` — the broker owns slow-consumer policy now.
   Net deletion: the entire timer dance (lines 491-495 of old
   code, plus the select arm at line 522) is gone.

5. **TUI dispatcher** at lines 555-573: `app.Subscribe(program)`
   now does its own `events := app.events.Subscribe(tuiCtx)` at
   line 558 and reads `ev.Payload` (line 569) instead of `msg`
   directly.

6. **Cleanup** at line 487: `app.events.Shutdown()` added to the
   cleanup func. Without this the broker would leak its
   subscriber goroutines on app shutdown.

7. **`checkForUpdates`** at line 626-630: rewired from raw send
   to `app.events.Publish(pubsub.UpdatedEvent, UpdateAvailableMsg
   {...})`.

Tests rewritten too:
- `TestSetupSubscriber_NormalFlow` (app_test.go:14-46) now uses
  a real `pubsub.Broker[tea.Msg]` as `out`, subscribes a channel
  for assertion, publishes two events, drains.
- `TestSetupSubscriber_SlowConsumer` from old code is deleted
  (no longer meaningful — broker handles backpressure).
- `TestEvents_ZeroConsumers` (lines 87-110) is the right
  regression for "publish must not block when nobody is
  listening" — that's the foundational invariant of the broker.

## Risks

- **API break**: `Events()` → `Events(ctx)` and the element type
  change. Anyone embedding crush as a library has to update
  call sites. Acceptable: the old API was buggy by design.
- **Memory**: a never-cancelling subscriber will accumulate
  events in its broker buffer. The `Subscribe(ctx)` API ties
  lifetime to context, which is the right contract — but a
  caller that passes `context.Background()` and forgets to
  consume will leak. The broker presumably has a per-subscriber
  bound; worth verifying in `pubsub.Broker.Subscribe`.
- **Event ordering across subscribers**: the broker preserves
  order *within* a subscriber but offers no cross-subscriber
  ordering guarantee. For SSE clients this is fine; if any
  consumer was relying on global ordering it would have been
  broken anyway.
- The `tea.Msg(event)` cast at line 504 is structurally fine but
  worth a comment — `pubsub.Event[T]` is a struct, casting it
  to `tea.Msg` (an `any`-alias) erases type info. The TUI side
  unwraps `ev.Payload` to recover. Working as intended; just
  non-obvious.

## Suggestions

None blocking. Nit: the deleted `subscriberSendTimeout` constant
(old line 489) had a 2-second value documented inline; if the
new broker exposes a similar knob, mention it in the PR body so
operators know where the equivalent backpressure setting lives.

## What I learned

The "shared channel becomes implicit competing-consumer" bug is
one of those Go traps that looks correct in code review and only
manifests when N > 1 consumers exist. The pattern fix —
"channels are point-to-point; brokers are pub-sub" — is the
right mental model. Worth leaving a comment near the broker
field saying *why* it's a broker and not a channel, so the next
person doesn't "simplify" it back into a `chan`.
