# BerriAI/litellm PR #27081 — chore(proxy): close callback-config and observability-credential side channels

- Author: stuxf
- Head SHA: `01323e890357f34ea39eab025e7fff55391ed26e`
- Verdict: **merge-as-is**

## Rationale

Substantive proxy hardening that closes two distinct exfil primitives.
First, `litellm/proxy/auth/auth_utils.py:175-281` derives the banned
observability params from the canonical `_supported_callback_params`
allowlist (minus a small explicit `_SAFE_CLIENT_CALLBACK_PARAMS`
exception set for `langfuse_prompt_version` / `langsmith_sampling_rate`),
so any new integration registered upstream is banned-by-default — the
exact failure mode you want. Second, `_NESTED_METADATA_KEYS` at line 181
plus the metadata walk at `:343-346` and the `_coerce_metadata_to_dict`
helper at `:351-368` plug the previously-uncovered path where
`metadata.langfuse_host` / `litellm_metadata: "{...}"` (form-data string
form) bypassed the root-only banned-param check. Third change at
`litellm/proxy/litellm_pre_call_utils.py:123-135` adds `callbacks`,
`service_callback`, `logger_fn`, `litellm_disabled_callbacks` to the
untrusted-fields list — these mutate the process-wide callback registries
in `litellm.utils.function_setup`, so one request poisoning subsequent
callers is a real concern. The per-param `continue` instead of `return` at
`auth_utils.py:294-298` is correctly justified by the inline comment.
Test additions confirm the new behavior. Ship.
