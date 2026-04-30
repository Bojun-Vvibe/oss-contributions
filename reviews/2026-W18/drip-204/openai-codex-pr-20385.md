# openai/codex PR #20385 — fix(login): unblock server on cancellation

- **URL:** https://github.com/openai/codex/pull/20385
- **Head SHA:** `5598368bdc484e2eb6c9ed28d5bff11ad617d6f4`
- **Diff size:** +8 / -2 in `codex-rs/login/src/server.rs`
- **Verdict:** `merge-as-is`

## Specific diff references

- `codex-rs/login/src/server.rs:124-128` — `ShutdownHandle` gains a second field `server: Arc<Server>`. The struct also drops `Debug` from the derive list (now `#[derive(Clone)]` only) because `tiny_http::Server` doesn't implement `Debug`. Losing the auto-derived `Debug` is a minor regression for anyone who was logging the handle, but `ShutdownHandle` is a control-plane type unlikely to appear in `{:?}` formatters; acceptable trade-off.
- `codex-rs/login/src/server.rs:133-137` — `ShutdownHandle::shutdown()` now calls `self.server.unblock()` *after* `notify_waiters()`. This is the actual fix: `tiny_http::Server::recv()` will block indefinitely until either a request lands or `unblock()` is invoked. Without this call, a `notify_waiters()` posted from one thread would only wake the async branch; the blocking `recv()` thread would sit there until the next inbound HTTP request, defeating cancellation entirely.
- `codex-rs/login/src/server.rs:188-190` and `:248-251` — `shutdown_server = server.clone()` is captured *before* `server` is moved into the worker thread, then plumbed into the returned `ShutdownHandle`. This is the correct ordering — `Arc::clone` is cheap and the borrow checker would otherwise reject the move-then-reference pattern. The new struct literal at `:249-251` wires both fields explicitly.

## Reasoning

The pattern being fixed (notify + unblock for a blocking listener wrapped in async cancellation) is a well-known footgun with `tiny_http`, and the patch is the canonical fix: hold an `Arc<Server>` on the cancel handle and call `Server::unblock()` alongside the `Notify`. Eight lines, one concept, zero behavior change in the success path. The contributor's testing notes call out a specific E2E that previously raced under cancellation (`falls_back_to_registered_fallback_port_when_default_port_is_in_use`), run three times consecutively to confirm flake elimination, plus a full `cargo test -p codex-login` and `just fix -p codex-login` — that's the right test set for a concurrency fix.

One thing worth flagging but not blocking: `Server::unblock()` is *advisory* in `tiny_http` — it nudges the listener to return early on the next loop iteration, but if `recv()` is already mid-`accept()` for a real connection the handler will still process it. For a login server that races a browser callback against an explicit cancel, that's fine (worst case is the cancel fires after the user already pasted the code and the success path wins). If the surrounding loop relies on `recv()` returning a sentinel error to break, double-check the loop body handles `IoError::WouldBlock` / unblock-induced returns gracefully — but the existing structure already worked with `notify_waiters`, so this is a no-op concern in practice.

Drop of `Debug` is the only API-surface change. The struct is internal to `codex-login` and constructed only from `run_login_server`, so external consumers shouldn't notice. Approve and merge.
