# BerriAI/litellm #27044 — fix: guard end_user_id or-fallback with disable_end_user_cost_tracking flag

- URL: https://github.com/BerriAI/litellm/pull/27044
- Head SHA: `df390267523616c6384631b2a51b3c8cad8dc6a8`
- Author: @xodn348
- Stats: +60 / -3 across 2 files (1 fix + 1 regression test)

## Summary

Bug fix for issue #27038: when an operator sets
`litellm.disable_end_user_cost_tracking=True` to keep PII out of spend
logs, the previous logic in `get_logging_payload` would still
back-fill `end_user` from `standard_logging_payload["metadata"]
["user_api_key_end_user_id"]` via the `or` chain. This PR gates that
fallback on the kill-switch.

## Specific feedback

- `litellm/proxy/spend_tracking/spend_tracking_utils.py:295-301` — the
  guard is correct: only the **standard_logging_payload** fallback is
  conditional. Direct `end_user_id` set earlier in the function still
  flows through because the user explicitly populated it — that
  mirrors the documented semantics of `disable_end_user_cost_tracking`
  (suppress *implicit* sourcing, not *explicit* sets).
- Worth a one-line comment above line 298 stating the contract
  explicitly so a future refactor doesn't tighten this back into a
  blanket suppression.
- `tests/logging_callback_tests/test_spend_logs.py:561-617` — clean
  regression test: sets the global, builds a realistic kwargs payload
  with `user_api_key_end_user_id` populated in BOTH the
  litellm_params metadata AND the standard_logging_object metadata,
  asserts `payload["end_user"] == ""`, and restores the flag in
  `finally`. Good hygiene.
- One coverage gap worth filling: there is no symmetrical test
  proving the fallback **still works** when the flag is False. A
  `test_end_user_fallback_runs_when_tracking_enabled` would lock the
  positive path.
- Defensive nit: `getattr(litellm, "disable_end_user_cost_tracking", False)`
  would be safer if a future stripped-down build of the litellm module
  ever omits the attribute; current code accesses it unconditionally
  and would `AttributeError`. Probably overkill — the attribute is
  declared in the top-level module — but worth a thought.
- No breaking change: callers already opted into this flag get the
  documented behaviour; everyone else is unaffected.

## Verdict

`merge-after-nits` — minimal, correct, regression-tested. Add the
positive-path test and ship.
