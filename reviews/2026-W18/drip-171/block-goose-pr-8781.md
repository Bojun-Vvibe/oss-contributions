# block/goose#8781 — `fix: handle acp requests concurrently`

- URL: https://github.com/block/goose/pull/8781
- Head SHA: `1a5934143d80`
- Author: (Goose contributor)
- Size: +173 / -123 in `crates/goose/src/acp/server.rs`
- Body: "this was slowing things down quite a bit"

## Summary

The ACP server's top-level dispatch (`HandleDispatchFrom<Client> for GooseAcpHandler`)
was awaiting each request handler inline inside the `Box::pin(async move {...})`
state machine, serializing all client-bound traffic on a single
connection. This PR converts most handler arms to
`cx.spawn(async move { ... })?` so they run concurrently, while
*deliberately* keeping `InitializeRequest` inline because it
mutates connection-scoped capability state that later handlers
depend on.

## Specific references

`crates/goose/src/acp/server.rs:3728-3760` (the `HandleDispatchFrom` impl):

- New explanatory comment at line ~3731 — well written and
  exactly the kind of guardrail the next refactorer will need:

  ```rust
  // InitializeRequest runs inline: it sets connection-scoped state
  // (client fs/terminal capabilities) that later handlers read with
  // defaults, so a pipelined NewSessionRequest must not race ahead of it.
  ```

- `NewSessionRequest` arm at line ~3748 is now spawned:

  ```rust
  let agent = agent.clone();
  let cx_clone = cx.clone();
  cx.spawn(async move {
      responder.respond_with_result(agent.on_new_session(&cx_clone, req).await)?;
      Ok(())
  })?;
  Ok(())
  ```

- `CancelNotification` arm at line ~3795 — same treatment for a
  notification that previously blocked the dispatch loop.
- `SetSessionConfigOptionRequest` arm starting around line ~3805
  (visible at +90 line range) is rewritten as a spawned future
  with the same matching logic; only the wrapping changes.

## Design analysis

This is the right shape for an ACP server: each request is an
independent unit of work and the dispatch loop must not block
on one slow handler while a fast cancellation queues behind it.
The pattern (`cx.spawn(...)?; Ok(())`) preserves backpressure
semantics through the runtime's spawn channel rather than the
dispatch loop.

The carve-out for `InitializeRequest` is the critical piece. Two
things to verify:

1. **All ordering-sensitive handlers stay inline.** Inspecting
   the diff, only `InitializeRequest` is left inline in the
   visible portion. It's worth grepping the rest of `server.rs`
   for any other handler that mutates connection-scoped state
   (e.g. authentication, capability negotiation, session
   creation that the rest of the impl blanket-spawns) to confirm
   nothing else needs the inline guarantee. `NewSessionRequest`
   in particular *creates* a session ID that subsequent
   notifications reference; if a `CancelNotification` for that
   session arrives before `NewSessionRequest`'s spawned future
   completes the session-table insert, the cancel will no-op
   silently.

2. **`responder.respond_with_result(...)?` inside a spawn.** The
   `?` now propagates inside the spawned future rather than back
   to the dispatch loop. If `respond_with_result` returns an
   error (channel closed, peer disconnected), it now silently
   terminates the spawn instead of bubbling up. This is probably
   fine — a closed responder channel means the client is gone —
   but it's a contract change worth a sentence in the PR body.

## Risks

- **Race on session lifecycle.** As above: pipelining
  `NewSessionRequest` then `CancelNotification`/`PromptRequest`
  on the same session ID becomes order-undefined under the new
  model. Whether ACP clients are allowed to do that is a spec
  question; if so, this needs a per-session in-order queue, not
  per-connection.
- **Handler-internal state.** Any handler that took a `Mutex`
  spanning the await point on the assumption that the dispatch
  loop serializes everything will now contend. None visible in
  the diff, but a `rg "lock\(\).await" crates/goose/src/acp/`
  audit is worth doing once.
- **Backpressure.** `cx.spawn` is fire-and-forget; an abusive
  client can now queue unbounded work. The previous serial
  model was implicit backpressure. Probably not exploitable
  inside a single Goose user's session, but worth tracking.
- **Tests.** Diff shows zero new tests. A concurrency PR with
  no concurrency test (e.g. two `NewSessionRequest`s in flight,
  assert both responders fire and both sessions exist) is
  asking for a regression in a year.

## Verdict

`needs-discussion`

The change is correct in shape and the inline carve-out for
`InitializeRequest` shows the author understood the ordering
constraint. But the per-session ordering concern (especially
`NewSessionRequest` → subsequent notifications for that session)
needs an explicit answer before this lands. If the answer is
"clients are required to await `NewSessionResponse` before
sending session-scoped traffic," then `merge-after-nits` with a
test added.

## Suggested nits

- Add a sentence to the PR body about the ACP ordering contract
  (or link to the spec section).
- Add a regression test that fires two long-running requests on
  the same connection and asserts both complete in parallel.
- Audit `crates/goose/src/acp/server.rs` for any remaining
  `*.lock().await` inside handlers that previously relied on
  serial dispatch.
- Consider whether `set_session_config_option` (the
  `SetSessionConfigOptionRequest` arm that already spawns a
  *background* refresh task via `tokio::spawn`) now races its
  own background work with subsequent inline spawns of the
  same handler.
