# BerriAI/litellm PR #27195 — fix: guard against model_info=None in router_strategy callbacks

- URL: https://github.com/BerriAI/litellm/pull/27195
- Head SHA: `f9645e51864ef67e9abfc1802cbe57edfbef92db`
- Author: aryamaddel (Arya Maddel)
- Files: `litellm/router_strategy/{lowest_cost.py,lowest_tpm_rpm.py,least_busy.py}` + tests (+139/-8)

## Assessment

This is a textbook one-line-per-callsite null-safety fix. The root cause analysis in the PR description is precise: `get_litellm_params()` at `litellm_core_utils/get_litellm_params.py:120` always packs `model_info: model_info` into the returned dict, including when `model_info` is `None`. So `dict.get("model_info", {})` returns `None` (the default branch only fires for *missing* keys, not for `None` values), and the subsequent `.get("id", None)` raises `AttributeError: 'NoneType' object has no attribute 'get'`. The fix `(kwargs["litellm_params"].get("model_info") or {}).get("id", None)` correctly handles both the missing-key and explicit-None cases.

The fix is applied at all 8 callsites: `lowest_cost.py:33,117`, `lowest_tpm_rpm.py:43`, and `least_busy.py:37,65,98,132,166`. The PR notes this matches the existing pattern already in `lowest_latency.py:53,206,277` — that's the right reference implementation to converge on, so this PR makes the four router-strategy modules internally consistent rather than fixing a problem in three of them and leaving a fourth incompatible.

The big concern the PR description itself flags is real and worth surfacing: the crash is silently swallowed by `except Exception: pass` at the outer level of each callback, which means production users have been seeing routing/cost-tracking silently degrade for any deployment whose `model_info` drops to `None` — and they have no log signal to even know. This PR fixes the AttributeError, but it doesn't add a log line for the `model_info is None` case, so a deployment in that state will still silently skip routing decisions without any operator visibility. A `verbose_router_logger.debug("model_info is None for model_group=%s, skipping routing", model_group)` inside each callback would close that gap. Not blocking — the immediate AttributeError fix is what's urgent — but worth a follow-up.

Test coverage is solid: 4 new tests per affected file (sync success, sync failure, async success, async failure) for `least_busy.py` and `lowest_cost.py`, plus a missing-key variant. That's 9-ish new test cases hitting the previously-crashing callbacks with both `model_info: None` and missing-key shapes. The existing 31-test suite still passes per the PR description. Tests are written as direct callback invocations with hand-crafted `kwargs` dicts which is the right level of unit isolation for this kind of defensive guard.

## Verdict

`merge-as-is`
