# sst/opencode#25017 — test: cover HttpApi websocket proxy

- PR: https://github.com/sst/opencode/pull/25017
- HEAD: `eb541952ecc4318bf0247a2af3be8e380f6a9235`
- Author: kitlangton
- Files changed: 1 (+71 / -12) — `packages/opencode/test/server/workspace-proxy.test.ts`

## Summary

Adds the first real WebSocket-path coverage for `HttpApiProxy.websocket`,
extending the existing HTTP proxy test file with (a) shared `listenServer` /
`listenTestServer` helpers that build the upstream into either the
test-injected layer or a fresh per-test scope, (b) an `echoWebSocket`
upstream that announces the negotiated subprotocol on `onOpen` then echoes
each frame, and (c) a new `it.live` case that wires
client → proxy listener → `HttpApiProxy.websocket` → upstream listener and
asserts both protocol forwarding (`protocol:chat`) and bidirectional message
relay (`echo:hello`). Adds `Socket.layerWebSocketConstructorGlobal` to the
shared layer so the WebSocket constructor is provided to every test in the
file uniformly.

## Cited hunks

- `packages/opencode/test/server/workspace-proxy.test.ts:11-16` — the new
  shared `testServerLayer` adds `Socket.layerWebSocketConstructorGlobal`
  alongside the existing `NodeHttpServer.layer`. Necessary because the
  Effect WebSocket primitives need a constructor in context; without it
  the new test would fail at construction-time with a missing-service
  error.
- `packages/opencode/test/server/workspace-proxy.test.ts:24-33` — the
  comment at `:27-28` ("Build into the current test scope so the listener
  stays alive until the test finishes. Using `Effect.provide` here would
  release it immediately") is the load-bearing decision: the upstream
  needs a longer scope than the proxy listener so the in-flight WS frames
  can complete after the proxy has finished its initial response.
- `packages/opencode/test/server/workspace-proxy.test.ts:35-49` —
  `echoWebSocket` does the right thing with `Effect.orDie` on
  `request.upgrade` (HTTP-not-upgradable is a programming error in this
  test path) and wraps both the `runRaw` body and the `onOpen` writer in
  `Effect.catch(() => Effect.void)` so a client disconnect on either side
  doesn't propagate as an unhandled defect.
- `packages/opencode/test/server/workspace-proxy.test.ts:133-152` — the
  new test's two assertions — `expect(yield* Queue.take(messages)).toBe("protocol:chat")`
  and `expect(yield* Queue.take(messages)).toBe("echo:hello")` — lock both
  the subprotocol-forwarding contract (that `Sec-WebSocket-Protocol`
  survives the proxy hop) and the bytes-bidirectional path. The
  `closeCodeIsError: () => false` opt-out is correct here: any close code
  is acceptable as long as the assertions ran first.

## Risks

- The test relies on Queue ordering between `onOpen`-fired protocol
  message and the first echoed frame. If the upstream's `onOpen` fires
  asynchronously and the client races a `write("hello")` before
  `onOpen` lands, `Queue.take` would return `echo:hello` first and fail
  with a confusing message-mismatch. In practice the `socket.runRaw(...)`
  fork at `:144-145` registers the consumer before the explicit `write`
  at `:147`, so the ordering is stable, but a `Queue.takeBoth` /
  set-membership assertion would survive future scheduler tweaks.
- The new `listenTestServer` deliberately escapes the test layer's scope
  via `Layer.build`. If a future refactor adds an early `Effect.die` after
  the server starts but before the test asserts, the listener won't be
  released until the outer scope closes — could leak a listening socket
  per failed test in CI. A `Scope.addFinalizer(server.close)` would close
  that.
- No test asserts the close-code propagation direction (proxy → upstream
  on client close, or upstream → client on upstream close). That's the
  next frame-loss class to fence.

## Verdict

**merge-as-is**

## Recommendation

Land it. Test coverage for the WebSocket path was a real gap (the existing
two HTTP cases were the only proxy assertions in the file) and the
`Deferred`-free design — using `Queue` ordering plus an upstream that
sends a deterministic `protocol:` preamble — is the right shape. Follow-up
PRs can add the close-propagation case and the SSL/upstream-down failure
modes.
