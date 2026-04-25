# BerriAI/litellm PR #26512 — bind RAG ingestion config to stored credential values

@997f5179 · base `main` · +394/-4 · author `yuneng-berri`

## Summary
When `ingest_options.vector_store` references a stored credential by `litellm_credential_name`, the resolver now lets the credential definition's `api_key` / `api_base` win over per-call overrides. Previously it only filled in *missing* fields, so a stale or attacker-supplied `api_base` in the call config would silently override the curated credential.

## What changed
- `litellm/vector_stores/.../config.py:71-92` (the `_load_credentials_from_config` patched block) — the merge changes from "only fill missing keys"
  ```python
  if key not in self.vector_store_config:
      self.vector_store_config[key] = value
  ```
  to "credential values always take precedence":
  ```python
  for key, value in credential_values.items():
      self.vector_store_config[key] = value
  if "api_base" in self.vector_store_config and "api_base" not in credential_values:
      del self.vector_store_config["api_base"]
  ```
- Bulk additions to `litellm/model_prices_and_context_window_backup.json` (model price table) — unrelated drift; should be a separate commit.

## Key observations
- Semantic flip is the right call for the security posture: a stored credential is the source of truth, otherwise the indirection has no integrity guarantee.
- The trailing `del self.vector_store_config["api_base"]` is the subtle bit — it strips a caller-supplied `api_base` only when the credential definition itself doesn't carry one. That handles the case where the credential is API-key-only and the caller tries to redirect traffic. Worth a one-line code comment so a future maintainer doesn't "simplify" it.
- The same logic should arguably apply to *all* fields, not just `api_base`. Why is `api_base` singled out? If a credential omits e.g. `aws_region`, should a caller still be able to set it? The asymmetry deserves either a comment or the same treatment for the other transport-shaping keys.
- Bundling ~150 lines of `model_prices_and_context_window_backup.json` churn into a security-flavored fix muddies the audit trail. Split.

## Risks/nits
- Behavior change for any caller that today relies on per-call `api_key` override beating the stored credential. That's a deliberate tightening, but flag it in the changelog.
- No test in the visible diff window for the new precedence rule — a 6-line `pytest` covering (a) credential overrides caller, (b) `api_base` stripped when absent from credential, would lock this in.

**Verdict: request-changes**
