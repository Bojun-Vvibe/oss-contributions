# sst/opencode #24720 — fix: prevent 100% CPU on Wayland by adding exponential backoff to SSE reconnect

- URL: https://github.com/sst/opencode/pull/24720
- Head SHA: `be1f828ccac67b1e1f565790f4195bb1d335bb9f`
- Files: 1 (`packages/app/src/context/global-sdk.tsx`)
- Size: +6 / −1

## Summary

Adds bounded exponential backoff to the global SDK's SSE reconnect loop. The
prior `await wait(RECONNECT_DELAY_MS)` constant-delay retry meant that on
Wayland (where the Electron renderer can hit a fast-fail-on-connect path
producing back-to-back errors with sub-ms turnaround inside the loop), the
process burned 100% CPU on a tight reconnect loop. The fix doubles the delay
each cycle up to a 10s ceiling and resets to the base on the first
successful connection.

## Specific references

- `packages/app/src/context/global-sdk.tsx:130-131` declares the per-loop
  `let reconnectDelay = RECONNECT_DELAY_MS` and `MAX_RECONNECT_DELAY_MS = 10000`
  inside the `start()` closure but *outside* the `while`-loop, so the backoff
  state survives across iterations (correct) but is freshly re-initialized
  on each `start()` call after a `stop()`/`start()` cycle (also correct —
  user-initiated restart shouldn't inherit a stale 10s delay).
- `:158` resets `reconnectDelay = RECONNECT_DELAY_MS` on the *first* event
  arrival path (right after the initial heartbeat reset, before the
  `for await (const event of events.stream)` consumer loop). This is the
  load-bearing line — a transient "connect, get one event, disconnect" cycle
  correctly resets backoff; a "connect, get nothing, disconnect" cycle does
  not, which is the right discrimination because a dead connection that
  handshakes but never delivers is still pathological.
- `:205-206` is the actual backoff: `await wait(reconnectDelay)` then
  `reconnectDelay = Math.min(reconnectDelay * 2, MAX_RECONNECT_DELAY_MS)`.
  Doubling-after-wait (not before) means the first retry uses the unmodified
  base delay, which preserves prior latency for the common single-blip case.

## Risk

Low. The only behavioral change for healthy reconnect cycles is that
sustained failure walks the delay up to 10s, which is the entire point.
The `oxlint-disable-next-line no-unmodified-loop-condition` comment at the
`while` is unchanged, so the existing graceful-exit semantics are preserved.

## Nits (non-blocking)

- Constants both inside the closure: `RECONNECT_DELAY_MS` is module-scope
  but `MAX_RECONNECT_DELAY_MS = 10000` is a magic number redeclared on every
  `start()` call. Hoist next to `RECONNECT_DELAY_MS` for symmetry and to
  make tuning a single-edit operation.
- No jitter — pure deterministic doubling. If many clients reconnect against
  the same backend after a deploy bounce, they synchronize at the 10s
  ceiling. Add `* (1 + Math.random() * 0.25)` to break the herd; cheap and
  standard for this pattern.
- No test. A `vi.useFakeTimers()` based test exercising "5 consecutive
  connect failures yields delays [base, 2×, 4×, 8×, 10s, 10s, ...]" would
  protect against future regressions like "let's also reset backoff in the
  catch handler" that would silently un-fix this.

## Verdict

`merge-after-nits` — correct fix, right shape, but the magic-number ceiling
and missing jitter are easy-win robustness adds.
