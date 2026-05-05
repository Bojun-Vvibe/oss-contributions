# openai/codex PR #21113 — Auto-deny MCP elicitations for Xcode 26.4 clients

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/21113
- Head SHA: `492df69aa1ebac2ad992b26ba82d7038eebfcff9`
- Size: +162 / -13 across app-server, codex-mcp, core

## Summary

Adds a per-thread "auto-deny MCP elicitations" flag plumbed from the
app-server client identity down into
`ElicitationRequestManager` so that a known-broken client family
(Xcode 26.4, which shipped before app-server MCP elicitation
requests were client-visible) silently gets a `Decline` response
instead of hanging or surfacing an unhandled prompt.

The hack is gated by a tight client-identity check:

```rust
fn xcode_26_4_mcp_elicitations_auto_deny(
    client_name: Option<&str>,
    client_version: Option<&str>,
) -> bool {
    client_name == Some("Xcode")
        && client_version.is_some_and(|version| version.starts_with("26.4"))
}
```

Defined twice — once in
`codex-rs/app-server/src/request_processors/thread_processor.rs:3376-3385`
and once in
`codex-rs/app-server/src/request_processors/turn_processor.rs:1135-1145`.

## What works

- The plumbing is honest: every entry point that already received
  `app_server_client_name` / `app_server_client_version`
  (`thread_resume`, `thread_fork`, plus the inner
  `set_app_server_client_info`) now also propagates the
  `mcp_elicitations_auto_deny` decision into
  `CodexThread::set_app_server_client_info`
  (`codex-rs/core/src/codex_thread.rs:218-228`) and on into
  `Codex::set_app_server_client_info`
  (`codex-rs/core/src/session/mod.rs:749-771`), which forwards it
  to `mcp_connection_manager.set_elicitations_auto_deny(...)`.
- The auto-deny is preserved across MCP connection-manager
  rebuilds: `codex-rs/core/src/session/mcp.rs:254-260` copies the
  current value onto the freshly-built `refreshed_manager` before
  swapping it in. Easy to forget; good catch.
- `ElicitationRequestManager::register_handler`
  (`codex-rs/codex-mcp/src/elicitation.rs:88-117`) reads the
  `auto_deny` flag *inside* the per-request closure and short-
  circuits to `ElicitationAction::Decline` before any
  approval-policy work — so the decline path is genuinely cheap and
  can't accidentally still prompt the user.

## Concerns

1. **Duplicate identity gate.** The same
   `xcode_26_4_mcp_elicitations_auto_deny` function is copy-pasted
   into `thread_processor.rs:3376` and `turn_processor.rs:1135`. If
   the version range needs to widen to `26.4.x` + `26.5.0-beta` or
   if a second buggy client is added, both copies will drift. Lift
   into a shared helper module (e.g.
   `codex-rs/app-server/src/client_compat.rs`) and re-export.

2. **Silent decline is a footgun for non-Xcode users.** Today the
   check is narrow, but the pattern ("look at client name/version,
   silently decline elicitations") is exactly the kind of thing
   that gets copy-pasted later without the same diligence. A
   one-line `tracing::debug!` inside the auto-deny branch in
   `elicitation.rs:97-104` ("auto-declining MCP elicitation for
   client X") would make the next on-call someone's life much
   easier.

3. **No test coverage on the gate predicate.** Trivial table-driven
   test:
   - `("Xcode", "26.4.0") -> true`
   - `("Xcode", "26.4.1-beta") -> true`
   - `("Xcode", "26.5.0") -> false`
   - `("Xcode", None) -> false`
   - `("VSCode", "26.4.0") -> false`
   That's the entire surface area of the bug class; locking it in
   costs nothing.

4. **`StdMutex<bool>` for `auto_deny`** (elicitation.rs:38, 51).
   Conceptually fine but `AtomicBool` would express intent better
   and avoid the `unwrap_or(false)` swallow on a poisoned mutex
   (lines 76–80, 99–103). The poisoning fallback today silently
   *re-enables* elicitations on a panicked thread, which is the
   *less safe* default for the buggy-Xcode case.

5. **`set_app_server_client_info` signature change is breaking** for
   any out-of-tree consumers that called the public method on
   `Codex` / `CodexThread`. They now must pass a `bool`. If those
   exist (extensions, tests in another crate), they'll need a
   cycle. Probably acceptable inside the workspace.

## Verdict

**merge-after-nits** — collapse the duplicated predicate into a
single helper, add a `tracing::debug!` on the auto-deny path, swap
`StdMutex<bool>` for `AtomicBool` (or at minimum fail-closed on
poisoning so the buggy-client case stays safe), and add the
4-line predicate test. Core design is sound and the cross-reset
preservation in `session/mcp.rs:254-260` is the kind of detail that
proves the author actually thought it through.
