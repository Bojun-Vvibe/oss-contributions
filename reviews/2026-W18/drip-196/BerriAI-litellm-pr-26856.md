# BerriAI/litellm PR #26856 — fix models access group leak into /models - issue #25550

- PR: https://github.com/BerriAI/litellm/pull/26856
- Head SHA: `0b3de67edb7b2767a7a8c5dbec102d63bdbfd362`
- Files touched: `litellm/proxy/auth/model_checks.py` (+50/-0), `tests/test_litellm/proxy/auth/test_model_checks.py` (+171/-0) (+221/-0, 2 files)

## Specific citations

- The bug class: a virtual key created with `models=["foo"]` against an access group "foo" that is later removed (or never defined) keeps the literal string `"foo"` in `key.models`. The current `_get_models_from_access_groups` only strips entries that ARE in the active `model_access_groups` dict, so the stale string falls through and `/v1/models` returns it as if it were a model id. The PR's reproducer in the description (curl `/v1/models` returning `{"id": "team-sales-api", "object": "model", "created": 1677610602, "owned_by": "openai"}`) is a real production-affecting leak.
- New helper `_is_unresolvable_model_identifier` at `model_checks.py:67-99` is the right shape for the gate. Six positive markers (`proxy_model_list` membership, `model_access_groups` key, `litellm.model_list_set` membership, contains `/`, contains `*`, `ft:` prefix) and the default is "unresolvable → drop". The `proxy_model_list`-membership branch is documented as the most-common preservation path for custom enterprise aliases (per nit-locked test `test_get_complete_model_list_keeps_custom_proxy_alias`).
- The filter is gated to `if key_models or team_models:` at `model_checks.py:248` — the proxy-admin path (no key/team models, sourced from authoritative `proxy_model_list`) is *not* filtered. This is the right scope: don't rewrite the admin view, only the per-key/team view where the leak originates.
- Test coverage at `tests/test_litellm/proxy/auth/test_model_checks.py:230-403` is parameterized across the eight relevant axes:
  - `:230` `_drops_stale_access_group_string` (key_models path) — the headline regression
  - `:251` `_drops_stale_access_group_string_team` (team_models path) — locks both entry points
  - `:267` `_keeps_active_access_group_expansion` — negative regression: active groups must still expand
  - `:300` `_keeps_provider_qualified_string` — `bedrock/very-new-model` survives
  - `:321` `_keeps_known_base_model` — known LiteLLM model id survives even when not configured on the proxy
  - `:339` `_keeps_custom_proxy_alias` — custom proxy alias `internal-assistant` (in `proxy_model_list` but NOT in `litellm.model_list_set`) survives — explicitly locks the proxy_model_list-membership branch
  - `:362` `_keeps_finetune_id` — `ft:gpt-4o:my-org:custom-suffix:abc123` survives
  - `:380` `_does_not_filter_proxy_admin_path` — when both key/team are empty, `arbitrary-named-model` from `proxy_model_list` passes through untouched (locks the gating predicate at `:248`)
- The PR body shows BEFORE / AFTER curl output against `ghcr.io/berriai/litellm:v1.82.3` vs from-source build, with the BEFORE response containing the leaked `team-sales-api` and AFTER returning `{"data": [], "object": "list"}`. This is exactly the right-shaped proof-of-fix for an information-leak class bug.

## Verdict: merge-as-is

## Rationale

Minimal-blast-radius patch (+50 LOC of source, +171 LOC of regression covering all 8 axes including the proxy-admin gating predicate). The helper is well-documented with the per-branch rationale inline in the docstring, the filter site is narrowly gated to only the key/team path where the leak originates, and the regression locks both the "drop stale" path and the five "keep legitimate" paths plus the "don't filter admin" path. The custom-proxy-alias test at `:339` is particularly well-chosen — it locks the `proxy_model_list`-membership branch which would otherwise be the most likely false-positive class (enterprise deployments routinely add aliases not in `litellm.model_list_set`).

The only stylistic observation (not blocking): the helper inverts the natural reading direction — `_is_unresolvable_model_identifier` returns `True` for "drop me", and the filter at `:251` is `if not _is_unresolvable...`. The negation chain is correct but a sibling-named `_is_resolvable_model_identifier` returning the inverse would let the filter read as `if _is_resolvable_model_identifier(m)` (no `not`). Pure cosmetics; the current shape is the standard way "filter predicate" helpers are written and the inline docstring at `:67-87` makes the semantics obvious.
