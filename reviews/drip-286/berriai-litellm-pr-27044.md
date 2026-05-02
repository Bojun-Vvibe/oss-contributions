---
repo: BerriAI/litellm
pr: 27044
head_sha: df390267523616c6384631b2a51b3c8cad8dc6a8
title: "fix: guard end_user_id or-fallback with disable_end_user_cost_tracking flag"
verdict: merge-as-is
reviewed_at: 2026-05-03
---

# Review: BerriAI/litellm#27044 — `fix: guard end_user_id or-fallback with disable_end_user_cost_tracking flag`

**Head SHA:** `df390267523616c6384631b2a51b3c8cad8dc6a8`
**Stat:** +61 / −3 across 2 files. Author: xodn348. Closes #27038.

## What it changes

In `litellm/proxy/spend_tracking/spend_tracking_utils.py:295–301`, the
existing `or`-fallback that repopulated `end_user_id` from
`standard_logging_payload["metadata"]["user_api_key_end_user_id"]` ran
unconditionally — defeating the `disable_end_user_cost_tracking` flag
because earlier code paths (`get_end_user_id_for_cost_tracking()`) had
already returned `None` to honor the flag, only for the fallback to
re-leak the ID into SpendLogs.

The fix wraps the fallback in a guard:

```python
if not litellm.disable_end_user_cost_tracking:
    end_user_id = end_user_id or standard_logging_payload["metadata"].get(
        "user_api_key_end_user_id"
    )
```

A new test `test_disable_end_user_cost_tracking_blocks_or_fallback` at
`tests/logging_callback_tests/test_spend_logs.py:561–617` constructs a
`get_logging_payload()` invocation with the flag set and metadata
containing `"user_api_key_end_user_id": "should-be-blocked"`, then
asserts `payload["end_user"] == ""`. The test correctly saves and
restores `litellm.disable_end_user_cost_tracking` in a `try/finally`
so it doesn't leak state across the test suite.

## Assessment

- This is the right fix in the right layer. The flag's contract is
  "don't track end-user costs"; an unconditional fallback that
  re-populates the ID was a clear bug, and the guard restores
  end-to-end consistency with `get_end_user_id_for_cost_tracking()`.
- The 3-line change is mechanically minimal — exactly what a defensive
  fix should look like. No refactoring snuck in.
- The test is well-constructed: realistic payload shape, asserts on
  the published `payload["end_user"]` field rather than internal
  state, and includes a useful failure message
  (`f"Expected end_user='' when disable_end_user_cost_tracking=True,
  got {payload['end_user']!r}"`). The `try/finally` flag-restore is
  the right pattern for a module-global flag.
- "All 10 existing spend-log tests continue to pass" matches what the
  PR description claims. The change is a strict additional guard on
  an existing branch; no behavior changes when the flag is at its
  default `False`.
- The `# BUG FIX:` comment immediately below (line 304) is unrelated
  prior context — left alone, correctly.

## Nits

None worth blocking on. The fix, the test, and the scope are all
exactly proportionate to the bug.

## Verdict

**`merge-as-is`** — minimal, correct, well-tested fix to a privacy/
data-leak bug. Ship.
