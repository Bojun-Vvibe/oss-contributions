# BerriAI/litellm#26763 — Fix typo in error logging (Excepton -> Exception)

- **Repo**: BerriAI/litellm
- **PR**: [#26763](https://github.com/BerriAI/litellm/pull/26763)
- **Head SHA**: `ffcdce8ee45239fd44ebf35aa248613970b27e9f`
- **Author**: lpo-pb1234
- **Diff stats**: +3 / −3 (1 file)

## What it does

Pure typo fix in three log strings inside `litellm/caching/dual_cache.py`:
`"LiteLLM Cache: Excepton async add_cache: ..."` → `"LiteLLM Cache:
Exception async add_cache: ..."`. Three sites, all in `except` blocks of
async cache mutations.

## Code observations

- `litellm/caching/dual_cache.py:129` — `verbose_logger.error` in
  `increment_cache`'s except branch. Typo fixed.
- `:359` — `verbose_logger.exception` in `async_set_cache`'s except.
  Typo fixed.
- `:386` — `verbose_logger.exception` in `async_set_cache_pipeline`'s
  except. Typo fixed.
- All three message bodies retain the `async add_cache: {str(e)}` shape
  even at the `increment_cache` and `async_set_cache_pipeline` sites,
  which means the existing log message *still* misattributes the error
  to `async add_cache` regardless of which actual function failed. The
  typo PR doesn't address that, but it's worth flagging — a future
  log-mining query (`grep "Cache: Exception async add_cache"`) will
  match all three callers indistinguishably.

## Risk

Zero. String-only change in error-path log messages. No callers depend on
the misspelled string (a quick grep across the codebase would confirm — if
any test or downstream observability tool was matching on `"Excepton"` it
would now silently break, but that would be an actively bad smell to begin
with).

## Verdict: `merge-as-is`

The fix is correct, minimal, atomic, and obviously not intended to ship
as-is in the original code. Approve and merge.

## Optional follow-up (separate PR)

Replace the misleading `async add_cache` substring at `:129`, `:359`,
`:386` with the actual function name (e.g.
`async_increment_cache`, `async_set_cache`, `async_set_cache_pipeline`)
so production log searches can isolate which cache mutation failed.
That's a behavior-affecting change that deserves its own PR with a
short note in the changelog about log-string churn for anyone with
ELK/Splunk dashboards keying on the old string.

## Follow-ups

- Run `rg -n "Excepton|Excpetion|Excpetion|Exceptn" --hidden` across
  the repo once to catch any siblings of this typo.
- Consider a `ruff` / pre-commit spell-check rule for log/exception
  string literals to prevent the next round.
