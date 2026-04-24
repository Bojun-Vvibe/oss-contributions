# BerriAI/litellm #26429 — fix: set verbose_logger level when LITELLM_LOG=INFO

**Link:** https://github.com/BerriAI/litellm/pull/26429
**Tag:** logger-parity, silent-default

## What it does

Fixes #26396. In `litellm/proxy/proxy_server.py::initialize`, the
`LITELLM_LOG=INFO` branch was setting `INFO` on
`verbose_router_logger` and `verbose_proxy_logger` but not on
`verbose_logger`. This meant `verbose_logger.info(…)` calls — used by
core (non-proxy, non-router) modules like the LLM call layer,
caching, callbacks — were silently suppressed at INFO even though the
operator opted into INFO. The corresponding `debug=True` and
`LITELLM_LOG=DEBUG` branches did set all three loggers; only the INFO
branch had the gap.

The fix imports `verbose_logger` alongside the other two and adds
`verbose_logger.setLevel(level=logging.INFO)`. One symmetric line.

## What it gets right

- **Minimal, surgical, parity-restoring.** The DEBUG branch's behavior
  becomes the contract (all three loggers escalate together); the INFO
  branch now matches. Future readers don't have to remember which
  branch handles which logger subset.
- Keeps the load-bearing comment
  `# this must ALWAYS remain logging.INFO, DO NOT MODIFY THIS`
  in place — clearly the level field is a tripwire for past regressions.
  The fix doesn't touch it.
- The fix is **observable**: operators who set `LITELLM_LOG=INFO` and
  weren't seeing `verbose_logger` output will immediately see the
  intended logs without changing their config. No migration step.

## Concerns / risks

- **Volume blast radius.** `verbose_logger` is the broadest of the
  three — it's used by request/response callbacks, model caching, the
  retry layer, and a lot of the `litellm.acompletion` hot path. Some
  deployments setting `LITELLM_LOG=INFO` may have been (incorrectly)
  relying on `verbose_logger` staying quiet, sized their log shipping
  for that, and will now see a step-function increase in log volume
  with no opt-out. A release-note callout is warranted.
- The PR doesn't add the inverse symmetry: if any new logger gets
  added later, the same three-line block in three branches has to be
  updated in lockstep. A small helper
  `_set_litellm_log_level(level)` that flips all three (and grows when
  a fourth logger appears) would prevent the next drift. This is the
  third time the gap has opened (DEBUG was added, then router/proxy
  were added, now this).
- No test. A unit test that imports `_logging`, calls `initialize`
  (or the relevant helper) with `LITELLM_LOG=INFO`, and asserts
  `verbose_logger.level == logging.INFO` would lock the parity going
  forward.
- `verbose_logger.setLevel` mutates a process-global logger — fine in
  the proxy server's startup path, but if `initialize` is ever
  re-entered (test harness, hot reload), the mutation order matters
  vs. any operator-overridden levels set between calls.

## Suggestion

Refactor the three-logger block in DEBUG and INFO branches to a single
helper `_apply_log_level(level)` that owns the canonical list of
loggers. The recurring drift is structural, not stylistic — every time
a new logger is added, two branches need editing. Centralize it.
