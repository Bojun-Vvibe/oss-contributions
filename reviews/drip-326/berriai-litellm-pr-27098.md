# BerriAI/litellm PR #27098 — fix(mcp): coerce YAML 1.1 string costs to floats; defend UI .toFixed (#27097)

- **PR:** https://github.com/BerriAI/litellm/pull/27098
- **Author:** Anai-Guo
- **Head SHA:** `8a65e68c` (full: `8a65e68c65d21d268a7ab83d13f14005c3e93866`)
- **State:** OPEN — closes #27097
- **Files touched:**
  - `litellm/proxy/_experimental/mcp_server/mcp_server_manager.py` (+142 / -4)
  - `tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server_manager.py` (+92 / -0)
  - `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_cost_config.tsx` (+25 / -24)
  - `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_cost_display.tsx` (+30 / -33)
  - `ui/litellm-dashboard/src/components/mcp_tools/types.tsx` (+14 / -0)

## Verdict

**merge-after-nits**

## Specific refs

- `mcp_server_manager.py:169-180` — `_coerce_optional_float` correctly rejects `bool` early (since `isinstance(True, int)` is True in Python, and `float(True) = 1.0` would silently succeed). That guard alone catches the most subtle path.
- `mcp_server_manager.py:185-220` — `_coerce_mcp_cost_info_in_place` mutates both `default_cost_per_query` and the `tool_name_to_cost_per_query` map, drops invalid entries, and emits a `verbose_logger.warning` with an actionable hint ("write `7.0e-5` instead of `7e-5`"). Drop-on-fail is the right call vs persisting bad values into JSONB.
- `mcp_server_manager.py:336-345` — applied at config-load time. Note the `dict(...)` copy before coercion — that prevents in-place mutation of the caller's `_mcp_info` reference, which is the right defense-in-depth move.
- `mcp_server_manager.py:222-252` — `_resolve_oauth2_flow` is **scope creep**. It infers `client_credentials` for legacy DB rows that have `token_url + client_id + client_secret` but null `oauth2_flow`. This is a separate bug from the YAML cost coercion and deserves its own PR + its own tests.
- `mcp_server_manager.py:2554-2640` — the `_call_regular_mcp_tool` changes (M2M-OAuth header gating, `has_client_credentials` branch) are also unrelated to the cost-coercion fix.
- `tests/.../test_mcp_server_manager.py:0-92` — 92 new test lines, but skimming the diff I see they cover the cost-coercion path. Need to confirm `_resolve_oauth2_flow` and the M2M header gating also have tests landed in this PR.

## Rationale

The cost-coercion fix is well-targeted: the YAML 1.1 footgun is real (`7e-05` parses as a string), the failure mode is user-hostile (UI crashes on `.toFixed` with no diagnostic), and the fix is defense-in-depth at exactly the right layer (config-load, before persist). The `bool`-rejecting `_coerce_optional_float` is a nice detail.

The blocker against merge-as-is is that this PR carries two unrelated changes: the OAuth2 flow resolver and the M2M Authorization-header gating. Both look reasonable but they're independent risk surfaces that shouldn't ride along on a UI-crash fix. Split into three PRs (cost coercion / OAuth2 flow inference / M2M header gating) and the cost one merges cleanly.
