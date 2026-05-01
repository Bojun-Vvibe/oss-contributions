# Review: BerriAI/litellm #26959 — fix: apply LITELLM_LOG to verbose_logger

- **PR**: https://github.com/BerriAI/litellm/pull/26959
- **Author**: Genmin (Joey Roth)
- **Head SHA**: `cd2d053f9bcf5418f83ed8aa081474fb81061af2`
- **Base**: `litellm_oss_staging`
- **Files**: `proxy/common_utils/debug_utils.py` (+7/-1),
  `proxy/proxy_server.py` (+6/-1),
  `tests/test_litellm/proxy/common_utils/test_debug_utils.py` (+72/-0)
- **Verdict**: **merge-as-is**

## Reasoning

Closes #26396. Real bug: `LITELLM_LOG=INFO` (and `=DEBUG`) only
adjusted the `verbose_router_logger` and `verbose_proxy_logger` log
levels, leaving the package-level `verbose_logger` at its default
`WARNING`. Result: users setting `LITELLM_LOG=INFO` to debug a problem
saw router and proxy info logs but not the package logs that often
carry the load-bearing diagnostic detail.

Two-site fix:

1. `debug_utils.py:808-841`: in the `INFO` and `DEBUG` arms of the
   `init_verbose_loggers()` env-var fallback, import `verbose_logger`
   alongside the existing two and call
   `verbose_logger.setLevel(level=logging.INFO)` /
   `setLevel(level=logging.DEBUG)`. The `DEBUG` arm also gets a small
   correctness nit: the trailing comment on the router-logger line
   was `set router logs to info` and is updated to `set router logs
   to debug`.
2. `proxy_server.py:5826-5840`: the equivalent env-var fallback path
   inside `initialize()` (not via `init_verbose_loggers`) gets the
   same `verbose_logger` import and `setLevel` call. Both fix sites
   are required because the codebase has two near-duplicate INFO
   handlers — one in the dedicated debug helper and one inline in
   `initialize()` — and the bug existed in both.

Test coverage at `tests/test_litellm/proxy/common_utils/test_debug_utils.py`:

- 72 lines, two tests. The first is a `parametrize`d unit test
  (`test_litellm_log_env_sets_all_verbose_loggers`) that checks both
  `INFO` and `DEBUG` env values produce the matching levels on **all
  three** loggers (`verbose_logger`, `verbose_router_logger`,
  `verbose_proxy_logger`) — the parametrize tuple correctly pins
  the expected levels.
- The second is an `asyncio` integration test that drives
  `proxy_server.initialize(debug=False, detailed_debug=False)` with
  `LITELLM_LOG=INFO` set, asserting the same three-logger contract on
  the second code path. This is the right depth — it's not just
  checking the helper, it's checking the production code path that
  includes the bug-site duplication.
- Both tests correctly snapshot the original logger levels in
  `original_levels` and restore them in a `finally` block, so test
  ordering can't pollute other tests in the suite.
- The `monkeypatch.setattr(banner, "show_banner", lambda: None)`
  trick to silence startup output during the integration test is a
  nice quality-of-life touch.

## Suggested follow-ups

- Real follow-up: the two `INFO`/`DEBUG` arms are now near-identical
  6-liners in two files. Worth a follow-up PR extracting
  `_apply_litellm_log_level(level)` to one place so the next time
  a fourth verbose logger is added (e.g. `verbose_callback_logger`),
  it's a one-line change in one file instead of four lines in two
  files.
- Optional: extend the parametrize to include `WARNING` and `ERROR`
  (or assert they no-op cleanly). Lock the contract that only `INFO`
  and `DEBUG` mutate verbose levels and other settings are inert.
