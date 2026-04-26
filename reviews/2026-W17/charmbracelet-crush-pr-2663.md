---
pr: 2663
repo: charmbracelet/crush
sha: b467763e87814f797891c05b1c5d08840ac2b8d6
verdict: merge-after-nits
date: 2026-04-26
---

# charmbracelet/crush #2663 — fix(app): replace single events channel with pubsub.Broker for fan-out

- **Author**: taigrr
- **Head SHA**: b467763e87814f797891c05b1c5d08840ac2b8d6
- **Link**: https://github.com/charmbracelet/crush/pull/2663
- **Size**: +157 / -152 across 4 files: `internal/app/app.go` (+16/-39), `internal/app/app_test.go` (+133/-107), `internal/backend/events.go` (+5/-3), `internal/server/proto.go` (+3/-3). Net code reduction in `app.go`.

## Scope

Real bug fix: SSE multi-consumer scenarios were broken because `App.events` was a plain `chan tea.Msg` with competing-consumer semantics — when multiple SSE clients subscribed to the same workspace, each received only a fraction of events. This PR replaces `App.events` with `*pubsub.Broker[tea.Msg]`, gives every caller of `Events(ctx)` / `SubscribeEvents(ctx, id)` its own dedicated channel via `Broker.Subscribe(ctx)`, and removes the old slow-consumer timer/drop logic in `setupSubscriber`.

## Specific findings

- `app.go:69` — `events chan tea.Msg` → `events *pubsub.Broker[tea.Msg]`. Type changes the public API contract on `Events()`. The signature changes from `Events() <-chan tea.Msg` to `Events(ctx context.Context) <-chan pubsub.Event[tea.Msg]`. **This is a breaking API change** — any external embedder that calls `app.Events()` without args, or that consumes the channel as `<-chan tea.Msg` directly, will fail to compile. The PR doesn't surface this in the title/description. If `App.Events` is part of an exported package surface used outside this repo, the deprecation path matters; if it's internal-only by convention, fine. Verify.
- `app.go:103` — `events: make(chan tea.Msg, 100)` → `events: pubsub.NewBroker[tea.Msg]()`. Buffer of 100 is dropped; the broker is unbuffered per-subscriber but the broker itself queues internally. Verify that `pubsub.Broker.Publish` is non-blocking under back-pressure (the test `TestEvents_ZeroConsumers` confirms it doesn't block with zero consumers; need to also verify it doesn't block under N slow consumers).
- `app.go:141-145` — `SendEvent` is now `app.events.Publish(pubsub.UpdatedEvent, msg)`. Old code was a `select { case ... default: }` non-blocking send with explicit drop. New code's blocking semantics depend on broker internals. The PR description claims "eliminating the slow-consumer timer/drop logic entirely" — confirm the broker has its own back-pressure handling (per-subscriber buffer + drop, or per-subscriber blocking with timeout). If it neither buffers nor drops, a single slow subscriber can stall the publisher and degrade the whole app.
- `app.go:486-492` — `cleanupFunc` adds `app.events.Shutdown()` after the existing `serviceEventsWG.Wait()`. Order matters: services are drained first, then the broker is shut down, which closes all subscriber channels. Correct.
- `app.go:496-509` (`setupSubscriber`) — deleted the timer machinery (`subscriberSendTimeout`, `time.NewTimer`, `Stop`/`Reset`/`<-C` drain dance). New body just forwards events into the broker. Net `-30` lines for the same correctness. Good simplification.
- `app.go:555-568` (`Subscribe(program *tea.Program)`) — TUI now does `events := app.events.Subscribe(tuiCtx)` then `program.Send(ev.Payload)`. The `tuiCtx` is the existing TUI lifetime context, so the subscription is auto-cleaned on TUI shutdown via the broker's context-driven unsubscribe. Correct.
- `app.go:626-633` (`checkForUpdates`) — old code did `app.events <- UpdateAvailableMsg{...}` (blocking send into a buffered channel). New code uses `app.events.Publish(pubsub.UpdatedEvent, ...)`. Same observable behaviour for consumers; same caveat about publisher back-pressure under slow consumers as `SendEvent` above.
- **Removed test `TestSetupSubscriber_SlowConsumer`** (`app_test.go:91-115` in the old file) — this test asserted that the old timer-drop logic dropped messages under a slow consumer. The new design's premise is "broker handles fan-out without dropping", so the test is correctly removed. **But**: there should be a NEW test asserting "publisher does not block when one of N consumers is slow" — the bug being fixed (multi-consumer fan-out) and the bug being prevented (slow consumer stalling publisher) are different correctness properties. The new tests cover fan-out (`TestEvents_NConsumers`, `TestEvents_OneConsumer`) and zero-consumers no-block (`TestEvents_ZeroConsumers`) but not slow-consumer-doesn't-block-publisher.
- **Removed test `TestSetupSubscriber_DrainAfterDrop`** and **`TestSetupSubscriber_NoTimerLeak`** — both tested the deleted timer logic. Correct to remove. The `goleak.VerifyNone` use is also removed; if the broker creates internal goroutines, a goroutine-leak test for the broker itself would be valuable but probably belongs in `internal/pubsub/`, not here.
- `app_test.go:1-14` — drops `testing/synctest` and `go.uber.org/goleak` imports. Both were specifically for the deleted tests. Clean removal.
- `app_test.go:18-50` (`TestSetupSubscriber_NormalFlow`) — replaces the synctest fixture with a straightforward broker-based fixture. Uses `time.Sleep(10 * time.Millisecond)` to yield so the subscriber goroutine can call `src.Subscribe` before publishing. This is timing-dependent — flaky on overloaded CI. Better: have the broker expose a way to detect "first subscriber attached" or use a `sync.WaitGroup` that the subscriber goroutine signals after subscribing. Minor flake risk.
- `app_test.go:52-78` (`TestSetupSubscriber_ContextCancellation`) — clean. Cancels context, waits for goroutine to exit with a 5s timeout. Good.
- `app_test.go:84-105` (`TestEvents_ZeroConsumers`) — publishes with no subscribers, asserts no block via a 1-second timeout. Validates the critical "broker doesn't block on empty fanout" property.
- `app_test.go:111-138` (`TestEvents_OneConsumer`) — publishes 10 events, asserts FIFO receipt with `tea.Msg(i)` value. Good.
- `app_test.go:144+` (`TestEvents_NConsumers`) — diff truncated at line 350; presumably asserts every subscriber receives every event (the actual fan-out fix). Need to confirm the test count and concurrency.
- `internal/backend/events.go` and `internal/server/proto.go` (+5/-3 and +3/-3) — call-site updates for the new `Events(ctx)` signature. Diff truncated; based on the line counts, these are mechanical signature passes-through.
- `SSE_MULTI_CONSUMER_BUG.md` — the PR mentions a writeup file is included. Including a markdown writeup of the bug in the PR is unusual; some maintainers prefer that lives in an issue or commit message rather than checked in. Worth confirming with @meowgorithm whether they want this file in the tree.

## Risk

Medium. The fix targets a real correctness bug (validated by SSE multi-client repro in PR description), and the diff is a net simplification (`-39 / +16` in `app.go`). Three residual risks:
1. **Public API change** on `App.Events()` signature — breaking for any external embedder.
2. **Publisher back-pressure semantics** under slow consumers depend on broker internals not exhibited in this PR.
3. **One timing-dependent test** (`TestSetupSubscriber_NormalFlow` 10ms sleep) — small flake risk.

## Verdict

**merge-after-nits** — fix is correct and the simplification is meaningful. Asks: (1) confirm whether `App.Events()` is part of an exported API consumers depend on and document the break, (2) add a "publisher doesn't block when one of N consumers is slow" test to lock in the new back-pressure contract, (3) replace the 10ms `time.Sleep` in `TestSetupSubscriber_NormalFlow` with a deterministic ready-signal, (4) clarify whether `SSE_MULTI_CONSUMER_BUG.md` belongs in the repo or in the PR description.
