# BerriAI/litellm #27143 — fix(security): prevent secret_fields from leaking into spend logs

- URL: https://github.com/BerriAI/litellm/pull/27143
- Head SHA: `b773a178cb5744e4737804cb00d33af268a8c1e4`
- Author: krrish-berri-2
- Size: +146 / -2 across 3 files

## Comments

1. `litellm/proxy/litellm_pre_call_utils.py:1409-1413` — Replacing `data["proxy_server_request"]["body"] = copy.copy(data)` with `_body_snapshot = {k: v for k, v in data.items() if k != "secret_fields"}` is the correct fix at the *source*. The previous line was the actual leak: it took a shallow copy of the entire request dict — including `secret_fields` which contains the raw `Authorization` header — and embedded it inside `proxy_server_request.body`, which then flowed into spend logs, lago, langsmith, and every other downstream logger. Good.
2. `litellm/proxy/litellm_pre_call_utils.py:1410-1413` — The added comment is excellent: it explains *why* the exclusion is needed (raw_headers contain Authorization tokens) and *where else* the leak would surface (spend logs, audit trails). Future maintainers will know not to revert this.
3. `litellm/proxy/litellm_pre_call_utils.py:1413` — Using a dict comprehension `{k: v for k, v in data.items() if k != "secret_fields"}` instead of `copy.copy()` produces a shallow dict, which is what the old code intended too. Behavior preserved for all non-secret keys; only `secret_fields` is dropped. Correct semantics.
4. `litellm/proxy/spend_tracking/spend_tracking_utils.py:613` — The `_SENSITIVE_REQUEST_BODY_KEYS = frozenset({"secret_fields"})` constant is the right shape for a defense-in-depth strip at the spend-log serialization layer. `frozenset` is the correct type (immutable, fast lookup). Future additions (e.g. `"x_internal_auth"`, `"raw_request_headers"`) just append to the set.
5. `litellm/proxy/spend_tracking/spend_tracking_utils.py:685-689` — Adding `if k not in _SENSITIVE_REQUEST_BODY_KEYS` to the existing `_sanitize_value` dict comprehension is the **correct belt-and-suspenders fix**. Even if a future code path bypasses the pre-call snapshot fix and lets `secret_fields` reach the sanitizer, this layer will strip it. Two layers of defense for a credentials-leak bug is exactly right.
6. `tests/test_litellm/proxy/spend_tracking/test_spend_tracking_utils.py:1526-1546` — `test_sanitize_request_body_strips_secret_fields` directly asserts the sanitizer drops `secret_fields` while preserving `model` and `messages`. Good unit test. The fake bearer token (`sk-GjX--WwRQmiX2cvASbKf5Q`) is obviously fake — good (don't want a real-looking secret in the test corpus).
7. `tests/test_litellm/proxy/spend_tracking/test_spend_tracking_utils.py:1549-1590` (likely continues past the diff window) — The end-to-end test name `test_proxy_server_request_payload_excludes_secret_fields` covers the integration path. Worth verifying it actually exercises `add_litellm_data_to_request` end-to-end and not just the sanitizer in isolation; if the latter, it's redundant with the previous test.
8. Missing: a test for the *negative* case at the source — call `add_litellm_data_to_request` with a `data` dict containing `secret_fields={"raw_headers": {"authorization": "Bearer sk-test"}}` and assert that `data["proxy_server_request"]["body"]` does not contain the string `"Bearer sk-test"` anywhere (recursive search). That's the integration test that proves the *original* leak is plugged.
9. Backport / disclosure: this is a credentials-leak fix. Confirm with security team whether (a) prior versions need a CVE / advisory, (b) users running affected versions should rotate their virtual-key bearer tokens. The fix is silently merged in this PR; the *disclosure* needs a separate channel.

## Verdict

`merge-after-nits`

## Reasoning

Correct two-layer fix for a real credentials-leak bug, with good comments explaining the *why* at both layers and three solid tests. The sanitizer-layer defense-in-depth is exactly what you want for a security fix. Two requests before merge: (1) add an integration test that asserts a fake bearer token never appears in the serialized `proxy_server_request.body` (the negative-search-on-string-match pattern); (2) coordinate with security for a CVE / advisory and a "rotate your keys" note in the release notes — silent fix isn't appropriate for a credentials-in-logs bug.
