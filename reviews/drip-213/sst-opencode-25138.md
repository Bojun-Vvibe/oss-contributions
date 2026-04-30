# Review: sst/opencode#25138 — Use PTY service directly in HTTP routes

- PR: https://github.com/sst/opencode/pull/25138
- Head SHA: `8f7f416261dd3c8c2ea3dc7ccbd7139eded9fedf`
- Files: `packages/opencode/src/pty/index.ts` (+30/-35), `packages/opencode/src/server/routes/instance/httpapi/handlers/pty.ts` (+5/-11)
- Verdict: **merge-after-nits**

## What it does

Two-part cleanup of the PTY HTTP-API handler boundary:

1. Drops the `EffectBridge.promise` round-trip that was wrapping `pty.create(...)` inside
   the `PtyHttpApi.create` handler at `routes/.../pty.ts:25-34`. The handler is already
   inside an `Effect.fn` generator, so `yield* pty.create(...)` is the canonical call shape
   and the bridge was redundant indirection.
2. Removes the `Instance.bind(...)` wrapper around the two long-lived `proc.onData(...)` /
   `proc.onExit(...)` callbacks registered inside `Pty.layer` at `pty/index.ts:230-263`.
   These callbacks now run as plain functions; `bridge.fork(bus.publish(...))` and
   `bridge.fork(remove(id))` are still used inside the exit handler, which is the path
   that actually needs Effect context.

## Notable changes

- `routes/.../pty.ts:1` — removes the unused `EffectBridge` import that previously fed
  the bridge call inside `create`.
- `routes/.../pty.ts:25-34` — the handler shrinks from a 9-line `EffectBridge.make()` →
  `Effect.promise(() => bridge.promise(pty.create(...)))` adapter down to a direct
  `yield* pty.create({ ...ctx.payload, args: ..., env: ... })`. The `args` / `env`
  defensive copies (`ctx.payload.args ? [...ctx.payload.args] : undefined`) are
  preserved exactly — the cleanup is structural-only on the call boundary.
- `pty/index.ts:5` removes the now-dead `Instance` import.
- `pty/index.ts:230-263` — the body of both callbacks is byte-identical to the prior
  `Instance.bind(...)` argument: same `session.cursor += chunk.length`, same subscriber
  fan-out loop with `readyState !== 1` and `sock(ws) !== key` early-cleans, same
  `BUFFER_LIMIT` ring-buffer truncation, same `session.info.status === "exited"` guard,
  same `bridge.fork(...)` publish-and-remove. Only the `Instance.bind` outer wrapper
  is gone.
- The PR description names two test files (`test/server/httpapi-pty-websocket.test.ts`
  and `test/server/httpapi-sdk.test.ts`) that pass post-change.

## Reasoning

The `Instance.bind` removal is the load-bearing claim. `Instance.bind` exists to thread
a per-instance request scope into a callback that would otherwise run outside the
ambient instance context. The `proc.onData` callback only mutates `session.*` state
(captured in the closure) and writes to subscriber WebSockets (also captured via the
`session.subscribers` Map); it does not look up anything from the instance scope. The
`proc.onExit` callback does call `bus.publish` and `remove(id)`, but both are wrapped
in `bridge.fork(...)` which is the explicit "fork into Effect context" boundary — so
removing the outer `Instance.bind` is safe iff `bridge.fork` itself preserves whatever
context those operations need. That is the existing pattern used elsewhere in this
file (see how `bus.publish(Event.Created, ...)` is yielded directly inside the
generator at `:280`), so the change is consistent with the surrounding code.

The `EffectBridge.promise` removal is straightforwardly correct — the handler is
inside `Effect.fn`, the call site is `yield*`-able, and the bridge was only there
because the handler was previously refactored to or from a non-Effect context.

## Nits (non-blocking)

1. `pty/index.ts:230-263` — no test exercises `proc.onData` and `proc.onExit` callback
   bodies through the layer construction path post-change. The PR cites WebSocket /
   SDK tests, which presumably cover the end-to-end behaviour, but the assertion that
   the cleanup is byte-identical relies on those tests catching subscriber-fan-out
   regressions. A unit test that registers a fake `proc`, drives `onData(chunk)`, and
   asserts subscribers receive the chunk + `session.cursor` advances would pin the
   refactor explicitly.
2. The PR title says "Use PTY service directly in HTTP routes" but ~70% of the diff is
   in `pty/index.ts`, not in routes. Title under-sells the scope; consider "Pty: drop
   Instance.bind and EffectBridge.promise round-trips" or similar.
3. `routes/.../pty.ts:25` — `Effect.fn("PtyHttpApi.create")(function* (ctx: { payload: ... })`
   is correct, but the function body now has a single `yield* pty.create(...)` statement;
   a follow-up could collapse the `function*` into a regular function returning the
   effect directly. Out of scope here.
4. No CHANGELOG / release-notes entry; this is a structural refactor with no user-visible
   behaviour change so that is probably correct, but worth confirming repo policy.
