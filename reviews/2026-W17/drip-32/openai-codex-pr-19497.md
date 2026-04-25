# Review: openai/codex#19497 — Streamline turn and realtime handlers

- **PR**: https://github.com/openai/codex/pull/19497
- **State**: DRAFT
- **Author**: pakrym-oai
- **Range**: +345 / −422
- **Head SHA**: `9de37de3c1cf6157c36f653a3317433056b55a80`
- **Base SHA**: `03c30b8825ea9c07c54240aec30c6695b1a6604a`
- **Verdict**: merge-after-nits
- **Reviewer date**: 2026-04-25

## What the PR does

Refactors `start_thread_turn` and the realtime equivalent in
`codex-rs/app-server/src/codex_message_processor.rs` (around lines 6280–6470)
from a long sequence of `if let Err(error) = ... { send_error; return; }`
guards into a single `async {}` block whose `Result<_, _>` is dispatched at
the end. Each step now uses `?` with `inspect_err` to attach analytics, and
a small `invalid_request` helper replaces the inline
`self.send_invalid_request_error(request_id, ...).await; return;` pattern.
This is the same slice the author has been doing across the
`pakrym-oai/streamline-*` chain (#19490–#19497).

## What I checked

- Behavioral equivalence on the input-limit branch: the original
  `validate_v2_input_limit` failure tracked
  `AnalyticsJsonRpcError::Input(InputError::TooLarge)` *and* sent an error
  response. The new version preserves both — `track_error_response` is
  called inside the `if let Err` arm before returning `Err(error)`, which
  the outer dispatch then sends. ✅
- Override conflict (`permissionProfile` + `sandboxPolicy`): old code sent
  `send_invalid_request_error` directly; new code returns
  `Err(invalid_request("..."))`. Verified `invalid_request` builds the same
  `JsonRpcError` shape so the on-wire payload is identical. ✅
- The `validate_turn_context_overrides` branch keeps the
  `format!("invalid turn context override: {err}")` wording — important
  because tests likely match on this string. ✅
- `submit_core_op` now bubbles via `?` into the same dispatch — the prior
  code already short-circuited on its error, so order of side effects
  (analytics → outgoing send) is preserved. ✅

## Nits / questions

1. **`inspect_err` vs `map_err`**: the `load_thread` and
   `set_app_server_client_info` arms use `inspect_err` to call
   `track_error_response` then re-throw with `?`. This is correct but
   inconsistent with the validation-limit arm above, which manually calls
   `track_error_response` *then* returns `Err(error)`. Worth picking one
   pattern for the whole function — `inspect_err` reads better.
2. **Dropped `Option<AnalyticsJsonRpcError>` argument**: the
   `track_error_response(... /*error_type*/ None)` calls litter the diff.
   Consider a thin `track_unclassified_error` wrapper to retire the
   `/*error_type*/ None` comment marker.
3. **Realtime handler symmetry**: the parallel `start_realtime_turn`
   refactor (lower in the file, ~line 6470) collapses identically — please
   verify the WebSocket close path still fires on `Err` since the outer
   dispatch was previously inline.

## Recommendation

Merge once the `inspect_err` / manual-track inconsistency in (1) is
unified. The semantic-equivalence story is clean and this is a clear net
reduction in branching; risk is bounded to the analytics-emission ordering
which I traced and is preserved.
