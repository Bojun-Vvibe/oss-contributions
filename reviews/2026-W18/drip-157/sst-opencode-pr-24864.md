# sst/opencode PR #24864 — fix: clear timeout after promise rejection

- **PR**: https://github.com/sst/opencode/pull/24864
- **Author**: Hona
- **Merged**: 2026-04-28T23:37:12Z
- **Head SHA**: `89bc279b05af`
- **Size**: +1/-2 across 1 file (`packages/opencode/src/util/timeout.ts`)
- **Verdict**: `merge-as-is`

## Context

`withTimeout(promise, ms)` is the workhorse `Promise.race` helper in
`packages/opencode/src/util/timeout.ts`. The previous implementation only
called `clearTimeout(timeout)` from a `.then(result => ...)` handler attached
to the inner promise — i.e. only when the inner promise *fulfilled*. When the
inner promise *rejected* before the timeout fired, the rejection won the race
(reject propagates synchronously through `Promise.race`), the caller saw the
real error, but the `setTimeout` handle was never cleared. Result: a stale
timer kept the event loop alive for up to `ms` more, and on the Bun/Node tick
when it eventually fired, it would invoke the rejector on an already-settled
race (a no-op for the consumer, but the original semantics were "leak a timer
on every rejected `withTimeout`").

## What changed

One-line semantic fix at `packages/opencode/src/util/timeout.ts:4-5`:

```diff
-    promise.then((result) => {
+    promise.finally(() => {
       clearTimeout(timeout)
-      return result
     }),
```

Switching from `.then(handler)` to `.finally(handler)` is exactly the right
primitive: `finally` runs on both fulfillment *and* rejection without
swallowing the value or the rejection reason, so the cleared-timer side
effect now happens on every settled outcome. The deleted `return result`
line is also correct — `.finally` deliberately ignores its callback's return
value and forwards the original settlement, so the explicit `return` was
already redundant before; removing it just stops misleading the reader into
thinking the value was being threaded.

## Why `merge-as-is`

- Smallest possible correct fix: the race arm and the rejection arm are
  unchanged; only the cleanup handler was wrong.
- Behavior on success is bit-identical: `.finally` returns a promise that
  resolves to the original fulfillment value, so the `Promise.race` arm still
  resolves to `result`.
- Behavior on rejection is now correct: timer is cleared, the original
  rejection propagates through `.finally`, race rejects with the inner
  reason.
- Behavior on timeout is unchanged: the inner promise's `finally` will still
  fire later (cleanup is idempotent — `clearTimeout` on an already-fired
  timer is a no-op), and the timer's reject path that already won the race
  is unaffected.
- No new code paths, no new error handling — just a primitive swap.

## Nits (non-blocking)

- A one-liner test could pin the regression: `await
  expect(withTimeout(Promise.reject(new Error("x")), 60_000))
  .rejects.toThrow("x")` followed by `expect(timers.pendingTimers).toBe(0)`
  in the test runner. Worth adding when this file next gets touched, not
  worth blocking on.
- The `let timeout: NodeJS.Timeout` is assigned inside the inner Promise
  executor and read from the outer `.finally` — relies on the executor
  running synchronously before any microtask, which is guaranteed by the
  spec. Worth a one-line comment if anyone ever revisits this file, but
  again not blocking.

## What I learned

`.then(value => { sideEffect(); return value })` looks innocuous but quietly
encodes "do this side effect *only* on success." Whenever the side effect is
"release a resource I acquired before kicking off this promise" (a timer
handle, an AbortController, a file descriptor, a span), the right primitive
is almost always `.finally`. The pattern is structurally identical to a
`try { ... } finally { release() }` block in synchronous code, and the same
"only-on-success cleanup is a leak" failure mode applies to both. Cheap,
unambiguous fix.
