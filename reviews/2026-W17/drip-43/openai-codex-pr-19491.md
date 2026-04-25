# openai/codex PR #19491 — Streamline account and command handlers

- **URL:** https://github.com/openai/codex/pull/19491
- **Author:** @pakrym-oai
- **State:** OPEN (base `pakrym/appserver-errors-plugins-apps-skills`, part of a stack)
- **Head SHA:** `35256b64e00f1b9371bc4e33dd563befac7abe75`
- **File:** `codex-rs/app-server/src/codex_message_processor.rs`

## Summary of change

Part of pakrym's app-server JSON-RPC error-handling refactor stack
(#19484 → … → #19491). Collapses the verbose `match
result { Ok => send_response + notifications, Err => send_error }`
boilerplate in the account/login/command handlers into:

```rust
let result = self.login_api_key_common(&params).await
    .map(|()| LoginAccountResponse::ApiKey {});
let logged_in = result.is_ok();
self.outgoing.send_result(request_id, result).await;
if logged_in { self.send_login_success_notifications(None).await; }
```

The `chatgpt` login path is similarly factored: `login_chatgpt_v2`
becomes a one-liner that calls a new `login_chatgpt_response()` ->
`Result<LoginAccountResponse, JSONRPCErrorError>`, with the post-login
notification block extracted to
`Self::send_chatgpt_login_completion_notifications(...)`.

## Findings against the diff

- **L1343–1358** (`login_api_key_v2`): clean reduction from ~25 lines to
  ~8. Captures `logged_in = result.is_ok()` *before* `send_result` moves
  the value, which is the right ordering. Behavior preserved: success
  still emits both `AccountLoginCompleted` (via the helper) and
  `AccountUpdated`.
- **L1430+** (`login_chatgpt_v2`): the new
  `login_chatgpt_response()` returns the `Err` path through
  `internal_error(format!("failed to start login server: {err}"))` —
  matches the prior `JSONRPCErrorError { code: INTERNAL_ERROR_CODE,
  message: …, data: None }` exactly. Good.
- **Background completion task**: previously inlined ~50 lines of
  notification + auth reload + `active_login` cleanup. Now delegates to
  `send_chatgpt_login_completion_notifications`. Need to verify the
  helper preserves the `active_login` reset semantics (the diff cuts
  off, but the structure shown — `let mut guard = active_login.lock()
  .await; if guard.as_ref().map(ActiveLogin::login_id) == Some(login_id)
  { *guard = None; }` — is moved verbatim outside the helper, which is
  correct since clearing the local `Mutex` should not be the helper's
  responsibility.
- **Stack dependency:** `baseRefName` is `pakrym/appserver-errors-plugins-apps-skills`,
  so this can't be merged independently. Reviewers should land the
  earlier PRs first (#19484, #19490) to keep the diff readable.

## Verdict

**merge-after-nits**

Refactor is purely mechanical and behavior-preserving. The only nits
worth raising in the upstream review:

1. Rename `logged_in` → `succeeded` in `login_api_key_v2` for symmetry
   with how the helper is named (`send_login_success_notifications`).
2. Consider making `send_chatgpt_login_completion_notifications` an
   `&self` method (vs. associated `Self::…`) to match the rest of the
   processor — the explicit `Arc` cloning in the call site is only
   needed because of the `tokio::spawn` move, not because the helper
   itself can't borrow `&self`. (Minor; arguable.)

No correctness concerns; no test changes needed because the public
JSON-RPC contract is unchanged.
