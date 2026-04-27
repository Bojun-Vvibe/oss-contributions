# BerriAI/litellm PR #26603 — `fix(proxy)`: include `verbose_logger` in `init_verbose_loggers` `LITELLM_LOG=INFO|DEBUG` branches

- **PR**: https://github.com/BerriAI/litellm/pull/26603
- **Author**: @xr843
- **Head SHA**: `1257fca2`
- **Size**: +8 / −0
- **Files**: `litellm/proxy/common_utils/debug_utils.py`

## Summary

Closes the remaining half of #26396. `init_verbose_loggers` has four startup branches: `debug=True`, `detailed_debug=True`, `LITELLM_LOG=INFO`, and `LITELLM_LOG=DEBUG`. The first two correctly import and `setLevel` all three loggers (`verbose_logger`, `verbose_router_logger`, `verbose_proxy_logger`); the LITELLM_LOG branches were importing and setting only the `router` and `proxy` loggers, leaving the package-level `verbose_logger` inheriting root's WARNING level and silently filtering out every `verbose_logger.info(...)` call (most notably from `token_based_routing`). The parallel half in `proxy_server.py` already landed; this PR closes the same hole in `common_utils/debug_utils.py` (taken when the proxy reads `WORKER_CONFIG` and the JSON omits `debug`/`detailed_debug`).

## Verdict: `merge-after-nits`

Correct, minimum-surface, follows the established four-branch pattern. Two small concerns: zero new tests and a confusing comment in the DEBUG branch.

## Specific references

- `litellm/proxy/common_utils/debug_utils.py:811-820` — INFO branch now imports and sets `verbose_logger` to `INFO`:
  ```python
  from litellm._logging import (
      verbose_logger,
      verbose_proxy_logger,
      verbose_router_logger,
  )

  verbose_logger.setLevel(level=logging.INFO)
  verbose_router_logger.setLevel(level=logging.INFO)
  verbose_proxy_logger.setLevel(level=logging.INFO)
  ```
  The preserved comment `# this must ALWAYS remain logging.INFO, DO NOT MODIFY THIS` is load-bearing — it's there because someone in the past tried to "tighten" these to WARNING and broke the user-facing log surface, so leaving it untouched is the right move.
- `litellm/proxy/common_utils/debug_utils.py:828-840` — DEBUG branch mirrors the INFO branch:
  ```python
  verbose_logger.setLevel(level=logging.DEBUG)
  verbose_router_logger.setLevel(level=logging.DEBUG)
  ```
  All three loggers now set to `DEBUG`. Symmetric with INFO.
- The fix shape is structurally identical to the already-landed `proxy_server.py` half, so the four-branch invariant ("all four branches set all three loggers") is now restored across both startup paths.

## Nits

1. The new comment at L815 says `# set package logs to info` while the existing comment two lines down still says `# set router logs to info` for the router logger (which is correct). For the DEBUG branch the new comment says `# set package logs to debug` but the existing router comment at L834 says `# set router logs to info` — that's a stale comment from before the DEBUG branch existed; the line itself sets DEBUG. Worth fixing the stale `# set router logs to info` to `# set router logs to debug` in the same PR since you're already touching the block.
2. No regression test. The PR's own test plan has three checkboxes left unticked (verifying the three startup-environment combinations actually produce visible `verbose_logger.info` records). A pytest fixture that monkey-patches `verbose_logger` and asserts `verbose_logger.level == logging.INFO` after `init_verbose_loggers()` runs in each of the four branches would lock the four-branch invariant against future regressions in either startup path.
3. The four near-identical branches (`debug` / `detailed_debug` / `LITELLM_LOG=INFO` / `LITELLM_LOG=DEBUG`) in `init_verbose_loggers` and the *same four branches* in `proxy_server.py`'s init path are screaming for a shared helper:
   ```python
   def _set_all_loggers(level: int) -> None:
       from litellm._logging import (verbose_logger, verbose_proxy_logger, verbose_router_logger)
       for lg in (verbose_logger, verbose_router_logger, verbose_proxy_logger):
           lg.setLevel(level)
   ```
   That collapses both files' branches and makes the next "we forgot logger X" bug structurally impossible. Out of scope for this hotfix, but file a follow-up.

## What I learned

When an init function has N parallel branches that each must perform M setup steps, the bug class "step k missing in branch j" recurs forever until the M steps are factored into a single call site. The fact that this exact bug had to be fixed twice (once in `proxy_server.py`, once here) on two consecutive PRs is the empirical signal that the abstraction is wrong, not that the contributors are inattentive. The right fix is structural: collapse the branches to a `_set_all_loggers(level)` helper so "did this branch handle all three loggers?" stops being a thing a reviewer has to verify.
