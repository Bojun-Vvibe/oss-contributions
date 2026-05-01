# BerriAI/litellm #26959 — fix: apply LITELLM_LOG to verbose_logger

- **PR**: https://github.com/BerriAI/litellm/pull/26959
- **Head SHA**: `08c70d324c78cac474f9b7cb19dab2526a2f9fdc`
- **Files reviewed**: `litellm/proxy/common_utils/debug_utils.py`, `litellm/proxy/proxy_server.py`, `tests/test_litellm/proxy/common_utils/test_debug_utils.py`
- **Date**: 2026-05-01 (drip-230)

## Context

LiteLLM exposes three module-level loggers:

- `verbose_logger` — package-wide (covers `litellm/llms/*`, `litellm/main.py`, `litellm/cost_calculator.py`, etc.)
- `verbose_router_logger` — Router instance lifecycle / fallback / retry decisions
- `verbose_proxy_logger` — proxy HTTP layer / auth / DB

The `LITELLM_LOG=INFO|DEBUG` env var is the documented way for proxy operators to crank
up logging without code changes. The bug closes a real operator surprise (#26396):
`LITELLM_LOG=INFO` updated only `verbose_router_logger` and `verbose_proxy_logger` but
**not** `verbose_logger`, so the package-level INFO output (which is where most provider
adapter / cost-calculator messages live) stayed at WARNING. To get those, the operator
had to set `debug=True` programmatically, which isn't the documented escape hatch for
production proxies.

## Diff

Two parallel call sites — `init_verbose_loggers()` at
`litellm/proxy/common_utils/debug_utils.py:808-840` and the
`litellm_log_setting.upper() == "INFO"` arm of `proxy_server.py:5826-5841`:

```diff
  from litellm._logging import (
+     verbose_logger,
      verbose_proxy_logger,
      verbose_router_logger,
  )

  # this must ALWAYS remain logging.INFO, DO NOT MODIFY THIS

+ # sets package logs to info
+ verbose_logger.setLevel(level=logging.INFO)
  verbose_router_logger.setLevel(level=logging.INFO)
  verbose_proxy_logger.setLevel(level=logging.INFO)
```

Same shape for the DEBUG arm at `debug_utils.py:825-845`. The trailing comment for the
router logger also gets corrected from `# set router logs to info` (cargo-culted from the
INFO arm) to `# set router logs to debug`.

## Observations

1. **Correct hole closed in the documented contract.** The README/docs for
   `LITELLM_LOG=INFO` describe it as "set log level for the proxy"; an operator
   reasonably expects that to include the package logs they see in their `WARNING`
   stream. The fix matches the documented behavior.

2. **Both INFO arms updated.** There are two INFO branches — one in
   `init_verbose_loggers()` (called via `setup_logging`) and one in `initialize()`
   (called via the proxy startup `cli/main.py` path). Either can run depending on
   how the proxy is launched (`litellm --config` vs. uvicorn-direct `proxy_server:app`).
   Diffing both is the right call; missing one would leave the bug live for half the
   launch shapes.

3. **`verbose_logger.setLevel(...)` is the correct API.** `verbose_logger` is a
   stdlib `logging.Logger` at `litellm/_logging.py`; `setLevel` is the per-instance
   level. The PR doesn't touch handler levels, which is correct — handlers attached to
   `verbose_logger` already respect the propagation chain.

4. **Test arm is correct.** `tests/test_litellm/proxy/common_utils/test_debug_utils.py`
   parametrizes `("INFO", logging.INFO), ("DEBUG", logging.DEBUG)` and asserts all three
   loggers' `.level` attribute equals the expected sentinel. The test correctly **resets
   levels in a `finally` block** (capturing `original_levels` at `:31`, restoring at
   `:48-49`) so it can run in any order without polluting other test modules' assumed
   logger state. Important detail.

5. **`init_verbose_loggers()` reads `WORKER_CONFIG`** to honor the
   `debug=False, detailed_debug=False` short-circuit; the test sets that env var to a
   minimal JSON before the call so the env-driven fallback at the bottom of the function
   actually fires. Without that setup the test would no-op because `init_verbose_loggers`
   would early-return on `WORKER_CONFIG.debug=True`.

## Nits

- The two INFO blocks at `debug_utils.py:808-820` and `proxy_server.py:5826-5841` are
  near-identical 8-line copies; folding them into a single
  `_apply_litellm_log_level(level)` helper in `_logging.py` would let the next "we missed
  another logger" PR be a one-liner. Not blocking.
- `# this must ALWAYS remain logging.INFO, DO NOT MODIFY THIS` is a good
  load-bearing-invariant comment but it lives between the `import` and the first
  `setLevel` call, ambiguous about *which* setLevel it pins. Move it directly above the
  `verbose_logger.setLevel(level=logging.INFO)` line so the next code-mover knows what
  not to refactor.

## Verdict

merge-after-nits
