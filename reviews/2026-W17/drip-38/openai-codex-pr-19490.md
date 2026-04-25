# openai/codex #19490 — Streamline plugin, apps, and skills handlers

- **Repo**: openai/codex
- **PR**: [#19490](https://github.com/openai/codex/pull/19490)
- **Head SHA**: `244a3ff32f67a357b9f8c543f05c31f5cd198cbb`
- **Author**: pakrym-oai
- **State**: OPEN (+329 / -587)
- **Verdict**: `merge-as-is`

## Context

One of a series of "streamline X handlers" slices that are converting
JSON-RPC handlers in `codex-rs/app-server/src/codex_message_processor.rs`
from the old `send_error / return; send_response` pattern to a
`Result<*Response, JSONRPCErrorError>` + single-call-site `send_result`
pattern. Net diff is overwhelmingly negative because every handler had
4-6 nearly-identical 7-line `JSONRPCErrorError { code, message, data:
None }` literal blocks.

## Design

Three structural pieces.

1. New error helpers imported at the top
   (`codex_message_processor.rs:7-9`):
   `internal_error`, `invalid_params`, `invalid_request`. These
   wrap the literal struct construction. Eight-line error blocks
   collapse to a one-line `Err(internal_error("failed to load app
   lists"))` (line 6587).
2. Each handler is split into a thin outer fn that owns `request_id`
   and a `Result`-returning inner fn that owns the work. Example
   `thread_unsubscribe` (lines 6384-6420): outer awaits the inner
   then `outgoing.send_result(request_id, result)` once. `?` replaces
   `match { Err(e) => { send_error; return; } }` blocks.
3. `apps_list` (lines 6512-6660) is the most-changed: error paths
   for cursor parse, `recv()` close, timeout, `Accessible(Err)`,
   `Directory(Err)` all collapse to `Err(...)`. `paginate_apps`
   already returned `Result<AppsListResponse, JSONRPCErrorError>`,
   so the bottom of the loop just becomes `return apps_list_helpers::
   paginate_apps(...)` (line 6651) instead of a match-then-send.

Notable: `send_internal_error` (line 6275-6283) is moved up but
otherwise identical, and the standalone `send_marketplace_error`
helper is deleted entirely (lines 6306-6334 in the old code) —
its callers now use `?` against helpers that already produce the
right `JSONRPCErrorError` variant. That's the right call: one
fewer indirection layer.

The change to `send_app_list_updated_notification(&outgoing, ...)`
→ `send_app_list_updated_notification(outgoing, ...)` (lines 6585,
6611) reflects that `outgoing` is already `&Arc<OutgoingMessageSender>`
in the new helper signature.

## Risks

- Pure refactor, behavior-preserving. The only observable change
  would be if `send_result` vs `send_response` differ on the wire,
  which a quick check of the outgoing-message machinery confirms
  they don't (`send_result` wraps the same JSONRPCResponse path).
- `skills_list` early-returns from the inside of a `for root in
  entry.extra_user_roots` loop (line 6692-6699). The map_err
  rewrite preserves the early-exit with `?`, which is correct, but
  it means a single bad path in one entry still aborts the whole
  list — same as before.
- `apps_list_response`'s loop calls `paginate_apps` and may early-
  exit before `accessible_loaded && all_loaded`. Reading the diff
  carefully: the `if accessible_loaded && all_loaded { return ...;
  }` is the only loop exit on success — so if the channel keeps
  yielding without setting one of those flags, the loop still
  spins. Pre-existing behavior, not introduced here.

## Suggestions

None. The refactor is mechanical, the helpers are at the right
level, and the line count win is real (587 → 329, ~44% reduction
on touched code).

## What I learned

This is the right way to refactor a handler module: lift error
construction into one-line helpers first, then convert call sites
to `Result + ?`, then collapse the outer fn to a 3-line shim. The
opposite order (try to convert handler bodies before extracting
helpers) leaves you reading the same `JSONRPCErrorError { code,
message, data: None }` block 60 times during review. Doing helpers
first means each handler diff is small and visually obvious.
