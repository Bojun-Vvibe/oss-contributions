# openai/codex #19495 — Streamline thread resume and fork handlers

- **Repo**: openai/codex
- **PR**: [#19495](https://github.com/openai/codex/pull/19495)
- **Head SHA**: `d8792fc640920033439884500d70017370fe0448`
- **Author**: pakrym-oai
- **State**: OPEN (+293 / -416, net -123)
- **Verdict**: `merge-as-is`

## Context

Part of the ongoing app-server JSON-RPC handler refactor (see #19484,
#19490, #19491, #19492, #19493, #19494, #19496, #19497, #19498 — same
author, same week). Pre-refactor, helpers like
`resume_running_thread` and the rollout-vs-history dispatch in
`thread/resume` emitted `send_error` directly inside the helper,
giving the call site five+ levels of nesting and two parallel error
paths (some helpers returned `Result`, others side-effected the
outgoing channel and returned `()`).

## Design

Single-file change in
`codex-rs/app-server/src/codex_message_processor.rs`. Three
canonical patterns repeat through the diff:

1. **Push `send_error` to the request boundary**. Helpers now return
   `Result<T, JSONRPCErrorError>` and the top-level handler does one
   `match` on the result, calling `self.outgoing.send_error(request_id, error).await`
   only at the boundary. Example: `resume_running_thread` at lines
   4298-4360 now returns `Result<bool, JSONRPCErrorError>` (line 137
   in the diff) where `Ok(true)` means "handled (already-running
   path)", `Ok(false)` means "fall through to fresh resume",
   `Err(e)` means "emit error and stop". The caller at lines 47-53
   collapses to a 7-line match.

2. **Hoist the rollout-vs-history fork** at lines 4093-4104 — instead
   of two parallel branches each owning their own `send_error`, the
   `if let Some(history) = history { ... } else { ... }` expression
   produces a single `Result<(ThreadHistory, Option<StoredThread>), _>`
   value, matched once.

3. **Replace inline closure error wrapping** with `?` in `await?`
   chains, e.g. `ensure_listener_task_running(...).await?` at line
   246, which previously was three lines of `if let Err(e) = ... { send_error; return; }`.

The error message strings are preserved verbatim (lines 19, 36,
150-152, 170-174, 200-202, 219-221, 275-276) — important because
those strings are part of the JSON-RPC client contract and a typo
here would be a silent regression in error UX. I spot-checked the
`cannot resume running thread … with mismatched path: requested {requested_path.display()}, active {active_path.display()}`
string at line 171-174 against the prior wording and it matches.

## Risks

- **None at the contract level**: identical inputs still produce
  identical `JSONRPCErrorError` payloads on the same code paths. The
  `(thread_history, resume_source_thread)` tuple shape at line 64 is
  preserved — both branches return the same shape, just consolidated.
- **One subtlety**: the `ApiVersion::V2` argument at line 244 is
  hard-coded in `ensure_listener_task_running`. This was already true
  pre-refactor in this codepath (the running-resume always V2), but
  it's now slightly more visible. Probably worth a comment if other
  versions ever come back.

Net -123 lines on a single hot file with no behavior change — this
is the kind of refactor that pays for itself the next time someone
adds an error case.

## Verdict

`merge-as-is` — clean mechanical refactor, error contract preserved,
fits the broader handler-streamlining series.
