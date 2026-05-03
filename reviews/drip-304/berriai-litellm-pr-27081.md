# BerriAI/litellm #27081 — chore(proxy): close callback-config and observability-credential side channels

- **PR:** https://github.com/BerriAI/litellm/pull/27081
- **Head SHA:** `37a22acf6f527870d4873c33c67a1078dc20f52b`
- **Author:** stuxf
- **Size:** +343 / -9 across 4 files

## Summary

Plugs two related bypasses in the proxy's request bouncer (`is_request_body_safe`):

1. The original check walked the request-body root and the `litellm_embedding_config` nested dict, but **not** `metadata` / `litellm_metadata`. Observability fields the proxy bans at the root (`langfuse_host`, `langsmith_*`, `arize_*`, `phoenix_*`, `wandb_*`, GCS, etc.) could be smuggled under those metadata containers and would still get picked up by the integration callbacks — a credential / data-exfil primitive.
2. A handful of fields read by integration callbacks (`posthog_api_url`, `phoenix_project_name`, `wandb_api_key`, …) weren't listed in `_supported_callback_params`, so the proxy didn't ban them at all.

## Specific references

- `litellm/proxy/auth/auth_utils.py:174-180` — new `_NESTED_METADATA_KEYS = ("metadata", "litellm_metadata")` constant. Comment clearly explains the bypass: "a value under `metadata.langfuse_host` redirects the same Langfuse client and leaks the same credentials as the root-level `langfuse_host`."
- Same file, around `:185-200` — `_SAFE_CLIENT_CALLBACK_PARAMS = frozenset({"langfuse_prompt_version", "langsmith_sampling_rate"})` carves out the *describe-the-request* fields (no destination / no creds), preserving legit client behavior. Good scoping.
- `:205-225` — `_EXTRA_BANNED_OBSERVABILITY_PARAMS` lists the callback fields not in the canonical allowlist (posthog/phoenix/wandb). Comment correctly flags the long-term cleanup ("fold these into `_supported_callback_params` so they share one source of truth") rather than silently working around the omission.
- `litellm/proxy/litellm_pre_call_utils.py` — call-site updated to walk the new metadata keys.
- `tests/test_litellm/proxy/auth/test_auth_utils.py` and `tests/test_litellm/proxy/test_litellm_pre_call_utils.py` — tests cover both nested-metadata banning and the safe/banned partition for the observability callback params.

## Concerns

- Allowlisting `langfuse_prompt_version` and `langsmith_sampling_rate` as "safe" is a judgment call — they don't choose destination, but `langsmith_sampling_rate=1.0` from a malicious client could amplify cost on a tenant-shared Langsmith account. Worth a brief note in the PR description that this is acceptable because the destination is still proxy-controlled.
- The comment block at `:205` already commits to a follow-up (folding into `_supported_callback_params`); a tracking issue link would make that contract enforceable.

## Verdict

**merge-after-nits** — this is a real security fix with clear comments and good test coverage. Nits: link a follow-up issue for the allowlist consolidation, and add a one-line note about why sampling-rate is on the safe list.
