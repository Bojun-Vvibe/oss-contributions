# BerriAI/litellm PR #27071 — chore(proxy): drop client-supplied pricing fields from request bodies

- **Repo:** BerriAI/litellm
- **PR:** #27071
- **Head SHA:** `25671105e22ee0afec79ddeb80d2db5f1ee5e3c3`
- **Author:** stuxf
- **Title:** chore(proxy): drop client-supplied pricing fields from request bodies
- **Diff size:** +340 / -0 across 2 files
- **Drip:** drip-294

## Files changed

- `litellm/proxy/litellm_pre_call_utils.py` (+57/-0) — adds `_CLIENT_PRICING_CONTROL_FIELDS = frozenset(CustomPricingLiteLLMParams.model_fields.keys())`, `_CLIENT_PRICING_METADATA_FIELDS = frozenset({"model_info"})`, the `_ALLOW_CLIENT_PRICING_OVERRIDE_METADATA_KEY` opt-in flag, helpers `_key_or_team_allows_client_pricing_override(...)` and `_strip_client_pricing_overrides(data)`, plus a new call site inside `add_litellm_data_to_request` (around `litellm_pre_call_utils.py:1166`).
- `tests/test_litellm/proxy/test_pricing_field_strip.py` (+283/-0) — full unit suite covering: every pricing field gets stripped, `metadata.model_info` and `litellm_metadata.model_info` both get cleaned, opt-in flag preserves the override, and request flow integration via `add_litellm_data_to_request`.

## Specific observations

- This is a security-relevant fix masquerading as a `chore:`. The comment block at `litellm_pre_call_utils.py:158-167` explains exactly why: `litellm.completion` accepts `input_cost_per_token` etc. as kwargs, and via `register_model` mutates the *process-wide* `litellm.model_cost` map — so a single malicious request could rewrite cost tracking for every subsequent caller in that worker. The PR title undersells this; consider re-titling as `fix(proxy):` or `security(proxy):` so it surfaces in the changelog.
- `litellm_pre_call_utils.py:163` — `_CLIENT_PRICING_CONTROL_FIELDS = frozenset(CustomPricingLiteLLMParams.model_fields.keys())` is exactly the right pattern: any new pricing field added to that Pydantic model is automatically covered. Excellent forward-compat.
- `litellm_pre_call_utils.py:248-266` — `_strip_client_pricing_overrides` walks both `metadata` and `litellm_metadata`, which matches the dual-naming pattern used elsewhere in this file. The `verbose_proxy_logger.debug(...)` line that names the dropped fields is operator-friendly: anyone debugging "why isn't my custom pricing being applied?" will find a breadcrumb.
- `litellm_pre_call_utils.py:1166-1167` — call site `if not _key_or_team_allows_client_pricing_override(user_api_key_dict): _strip_client_pricing_overrides(data)` is placed *after* the existing `_CLIENT_MOCK_CONTROL_FIELDS` loop and *before* the metadata sanitization at line ~1170. Ordering looks right — the pricing strip runs before any downstream consumer reads the data.
- The opt-in `allow_client_pricing_override` metadata key is keyed on the API key *or* the team metadata. That's consistent with `_key_or_team_metadata_flag_is_true` semantics elsewhere. Good.
- One gap: the strip operates on `data` keys at the top level, but the original docstring says pricing fields can also arrive via `extra_body` on OpenAI-compatible clients. Worth a quick check in `_strip_client_pricing_overrides` for `data.get("extra_body")` if that's a real path.
- Tests look thorough at 283 LOC. The opt-in path is exercised via both key-metadata and team-metadata, which is exactly right.
- Backwards compat: any existing operator who *was* relying on per-request pricing overrides will silently see those overrides ignored after this lands. The doc comment / debug log helps but a CHANGELOG entry calling this out would prevent surprise.

## Verdict: `merge-after-nits`

Strong fix with rigorous tests. Ask: (1) re-title from `chore:` to `fix:`/`security:` so it appears in changelog as a behavior change, (2) add CHANGELOG entry calling out the opt-in flag, (3) sanity-check whether `extra_body` carries pricing fields on the OpenAI-compatible path. None block.
