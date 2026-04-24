# PR #26397 — fix(proxy): add verbose_logger to LITELLM_LOG=INFO branch

- URL: https://github.com/BerriAI/litellm/pull/26397
- Author: Anai-Guo
- Head SHA: `acffbba3ba6165a1aa4dca52f31740f4271a3c0a`

## Summary

One-line behavioral fix in `proxy_server.initialize()`. The `LITELLM_LOG=INFO`
branch previously imported and set only `verbose_router_logger` and
`verbose_proxy_logger` to INFO. `verbose_logger` (used by the core package,
including `token_based_routing.py`) was left at the root logger's default
WARNING, so every `verbose_logger.info(...)` call in core was silently
suppressed when operators set `LITELLM_LOG=INFO`. Aligns the INFO branch
with the DEBUG branch's three-logger setup. Closes #26396.

## Specific callouts

- `litellm/proxy/proxy_server.py:5847-5862` — The fix is mechanical and
  symmetric with the existing DEBUG branch. Diff:
  ```python
  # was:
  from litellm._logging import verbose_proxy_logger, verbose_router_logger
  # now:
  from litellm._logging import (
      verbose_logger,
      verbose_proxy_logger,
      verbose_router_logger,
  )
  verbose_logger.setLevel(level=logging.INFO)  # set package log to info
  verbose_router_logger.setLevel(level=logging.INFO)
  verbose_proxy_logger.setLevel(level=logging.INFO)
  ```
  Correct.
- The preserved comment `# this must ALWAYS remain logging.INFO, DO NOT
  MODIFY THIS` sits above all three setLevel calls. That comment was
  written when only two loggers existed; please re-anchor it (or just
  attach the "ALWAYS remain INFO" clause to the new `verbose_logger.setLevel`
  line as well) so future drive-by refactors don't accidentally remove only
  the new line.
- Note duplicate fix: PR #26401 ("fix(proxy): set verbose_logger level when
  LITELLM_LOG=INFO") and PR #26429 ("fix: set verbose_logger level when
  LITELLM_LOG=INFO") cover the same three-line change. The maintainer
  should pick one and close the other two with a back-reference to the
  merged commit, otherwise the project will accumulate three identical
  PRs in the queue. (#26429 was already in INDEX through prior drip,
  meaning the maintainer is already triaging this; please coordinate.)
- No test added. This is hard to unit-test directly (logger level mutation
  has process-global side effects), but a smoke test that:
  ```python
  os.environ["LITELLM_LOG"] = "INFO"
  initialize(...)
  assert logging.getLogger("LiteLLM").level == logging.INFO
  ```
  would catch the regression-of-the-regression. Worth adding.
- The WARNING/ERROR/CRITICAL branches are explicitly out of scope (handled
  by referenced PR #24921). Confirm those branches similarly include all
  three loggers — if only the INFO/DEBUG branches were ever fixed, the
  WARNING branch likely has the same bug and the user-facing symptom will
  flip the next time someone sets `LITELLM_LOG=WARNING`.

## Risks

- None of significance. Logger levels have monotonic semantics here
  (turning INFO on is strictly more verbose), so the worst case is that
  operators see more log output than before — which is exactly what they
  asked for by setting `LITELLM_LOG=INFO`.
- Coordination risk: three duplicate PRs (this, #26401, #26429). Whoever
  merges first wins; the other two will conflict trivially.

## Verdict

**Verdict:** merge-as-is

Correct, minimal, symmetric with the DEBUG branch. Maintainer should
de-duplicate against #26401/#26429 (pick the one with the cleanest history)
and close the others.
