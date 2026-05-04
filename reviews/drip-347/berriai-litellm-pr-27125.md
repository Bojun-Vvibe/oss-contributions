# BerriAI/litellm#27125 — refactor(BaseAWSLLM): shared IAM cache and static-credential caching

- **Head SHA**: `0af69dc291cd038dbfabe444e8a520a435a6a907`
- **Verdict**: `merge-after-nits`

## Summary

Introduces a process-wide `_shared_iam_cache: ClassVar[DualCache]` on `BaseAWSLLM` and routes only static-credential flows through it. Bedrock passthrough constructs new instances per request, so the previous per-instance `DualCache()` was always cold. +157/-92 across `base_aws_llm.py`, `passthrough/transformation.py`, and a new test module.

## Findings

- `litellm/llms/bedrock/base_aws_llm.py:68-74`: `_shared_iam_cache: ClassVar[DualCache] = DualCache()` is created at class-definition time. This is the right default for a long-running proxy, but the docstring should call out that subclasses inherit the *same* instance (not a per-subclass cache). If a future subclass wants isolation it must shadow `_shared_iam_cache`, not just re-bind `self.iam_cache` in `__init__`.
- `:114-131` (`_get_or_set_cached_static_credentials`): correct shape — read-then-fill with the existing `get_cache_key` SHA256. The docstring explicitly excludes role-assumption / web-identity / profile / explicit-session-token flows from caching, which matches the LIT-2662 reference.
- `:212-227`: the old unconditional `cache_key = self.get_cache_key(args); _cached = self.iam_cache.get_cache(cache_key)` block is gone. Good — that was the subtle bug, since it cached refresh-bearing `Credentials` from STS flows under a key derived from the *input* args, masking expiry.
- `tests/test_litellm/llms/bedrock/test_base_aws_llm.py`: new unit coverage. Please confirm the tests assert *both* (a) static-key path hits the shared cache across two distinct `BaseAWSLLM()` instances and (b) an assume-role-shaped call does **not** populate the cache. The first is the regression guard; the second is the safety guard.
- Type imports `Callable, ClassVar` added at `:9-11` — fine.

## Recommendation

Land after a sentence in the `_shared_iam_cache` docstring about the inherited-instance gotcha and confirmation that the negative test (assume-role bypasses cache) exists. Behaviorally the change is sound and addresses a real perf regression in passthrough.
