# BerriAI/litellm#27029 — feat(spend-logs): suppress traceback in SpendLogs error_information row

- PR ref: `BerriAI/litellm#27029` (https://github.com/BerriAI/litellm/pull/27029)
- Head SHA: `87062f74ebc1c5a342b3576fb52d92b73d69c476`
- Title: feat(spend-logs): suppress traceback in SpendLogs error_information row
- Verdict: **merge-after-nits**

## Review

The new helper at `litellm/proxy/spend_tracking/spend_log_error_logger.py:1-82` is well
factored. The two-axis gate (`_is_suppression_env_enabled` + DEBUG-level override at
line 51) means operators who set `LITELLM_SUPPRESS_SPEND_LOG_TRACEBACKS=true` for prod
log volume reasons still get full tracebacks the moment they bump the logger to DEBUG
for an investigation — that's the right ergonomic. Reading the env var fresh on every
call (line 32) is also correct given how operators flip env vars at runtime via config
reloads.

The mass conversion in `db_spend_update_writer.py` is mostly mechanical and faithful:
the old `verbose_proxy_logger.error("...%s\n%s", ..., traceback.format_exc())` pattern
becomes `spend_log_error("...%s", ..., exc=e)` in roughly a dozen sites
(`db_spend_update_writer.py:196, 491, 538, 583, 624, 652, 705, 904, 1072, 1734`). The
replacement preserves the % format string semantics and now lets the logger module
attach `exc_info` properly, which is structurally cleaner than the manual
`traceback.format_exc()` interpolation.

The UI side at `proxy_track_cost_callback.py:62-66` blanks the
`error_information.traceback` field on the SpendLogs row when suppression is active —
exactly the "stop the traceback from polluting the per-row Metadata pane" outcome the
PR description targets. The companion tests at
`tests/test_litellm/proxy/hooks/test_proxy_track_cost_callback.py:478-513` exercise both
the default-on and env-set paths with a real raised exception (so `__traceback__` is
populated), which is the right way to test this.

Nits:
1. `litellm/proxy/utils.py:5106-5111` keeps the bare `verbose_proxy_logger.error(...)`
   for `update_daily_tag_spend` and adds a NOTE comment explaining why. Good — but the
   asymmetry will rot. Consider a follow-up that adds a `with_exc=False` kwarg to
   `spend_log_error` so this site can opt into the helper without the traceback
   regression, instead of carrying a special-case comment.
2. `spend_log_error_logger.py:73-74` calls `verbose_proxy_logger.error(message, *args,
   exc_info=True)` when `exc=None` — that relies on `sys.exc_info()` being populated,
   which is fine inside an `except` block but silently produces no traceback if the
   helper is ever called outside one. The test at line 138 covers the in-`except` case;
   worth one more test asserting graceful behavior outside.
3. The `traceback` import at the top of `db_spend_update_writer.py` should be removable
   now that all `traceback.format_exc()` call sites in this file are gone — verify with
   a linter pass before merge.
