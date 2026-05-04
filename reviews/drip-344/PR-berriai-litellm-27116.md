# BerriAI/litellm#27116 — fix(utils): OpenAI tool name sanitize + per-request restore

- PR ref: `BerriAI/litellm#27116`
- Head SHA: `cf7e71c83d3afefa1e328c332555f02ddcfff25f`
- Title: fix(utils): OpenAI tool name sanitize + per-request restore
- Verdict: **merge-after-nits**

## Review

The design choice here is correct and important: using `contextvars.ContextVar` for
the sanitized→original name mapping in
`litellm/litellm_core_utils/openai_tool_name_mapping.py:18-21` instead of a
process-wide cache is exactly what you want for an async server. A module-global dict
would have leaked names across concurrent requests and produced extremely confusing
"my tool got renamed to someone else's tool" bugs under load. The
`begin_openai_tool_name_mapping_scope()` reset at the top of every `completion()` call
(`litellm/main.py:1167`) gives each request a clean slate.

The provider gating in `_OPENAI_TOOL_NAME_VALIDATION_PROVIDERS`
(`openai_tool_name_mapping.py:24-39`) is the right policy — only sanitize for
providers that actually enforce the `^[a-zA-Z0-9_-]+$` rule. Restoring on the way
back in `convert_dict_to_response.py:74-84` and `streaming_chunk_builder_utils.py:299`
covers both the non-streaming and streaming paths, which is the kind of symmetry that
usually gets missed in a first pass.

Two nits before merge:

1. The bare `except Exception:` at `litellm/main.py:1180` swallows `get_llm_provider`
   failures and silently disables sanitization. This is probably the right behaviour
   for resilience, but it should at least log at debug level so the failure mode is
   discoverable when a user reports "my tool name has a dot and the request 400s".
2. `_make_unique_openai_tool_name` (`litellm/utils.py:7894-7906`) handles collisions by
   appending `_1`, `_2`, etc., but the suffix loop could theoretically run forever if
   somehow `used_names` already contains every truncation. In practice it can't (64
   chars × 10 digits is plenty), but a `for n in range(1, 10000)` bound + raise on
   exhaustion would be defensive.

Test coverage in `tests/test_litellm/test_utils.py` (+132 lines) is solid; verified
the round-trip restore is exercised. No correctness blockers.
