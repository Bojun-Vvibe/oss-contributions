# All-Hands-AI/OpenHands PR #14084 — redact API keys from MCP config logging

- **PR**: https://github.com/All-Hands-AI/OpenHands/pull/14084
- **Author**: @all-hands-bot (OpenHands Bot)
- **Head SHA**: `4d0cc28a2d9bd469060887551189a369aa6a9824`

## Summary

Four log call sites that previously emitted unredacted MCP server
config / URL / runtime config to the logger are wrapped:

1. `openhands/mcp/client.py:103-119` — `connect_http` error paths
   call `redact_url_params(server_url)` before formatting into both
   `error_msg` and `mcp_error_collector.add_error(server_name=...)`.
2. `openhands/runtime/action_execution_server.py:874-877` — the
   `update_mcp_server` info log now applies `sanitize_config(t)` on
   each dict tool spec (falling back to `redact_text_secrets(str(t))`
   for non-dict items).
3. `openhands/runtime/impl/remote/remote_runtime.py:248` —
   `_build_runtime` debug log now wraps `self.config` in
   `redact_text_secrets(str(...))`.
4. `openhands/runtime/mcp/proxy/manager.py:155-159` — adds an info
   log for "MCP Proxy config updated" using `sanitize_config(...)`.

## Verdict: `merge-after-nits`

Solid security-hardening PR. Each call site correctly chooses
between `redact_url_params` (for URL surfaces that may carry
`?api_key=...` or similar), `sanitize_config` (for dict configs that
have known-sensitive fields like `headers.Authorization`,
`env.API_KEY`), and `redact_text_secrets` (regex-based fallback for
freeform strings).

## Specific references

- `client.py:103-104` and `:114-115` — `safe_url = redact_url_params(server_url)`
  is computed once per branch and used for both the log message and
  the `mcp_error_collector` server name. Important because the error
  collector is later surfaced in the UI; using the *unredacted* URL
  there would leak through a second channel.
- `action_execution_server.py:874-877` — the list comprehension
  `[sanitize_config(t) if isinstance(t, dict) else redact_text_secrets(str(t)) for t in mcp_tools_to_sync]`
  is the right shape; previously the whole list was stringified
  through `redact_text_secrets` which is regex-based and may miss
  secrets buried inside dict values.
- `remote_runtime.py:248` — uses `redact_text_secrets(str(self.config))`.
  This is a regex pass over the stringified pydantic model; if the
  model has nested-dict secrets that don't match the regex (e.g.
  custom field names), they'll still leak. Switching to
  `sanitize_config(self.config.model_dump())` would be stronger.

## Concerns

1. **Two-tier defense inconsistency.** Three of four call sites use
   `sanitize_config` (the strong, schema-aware redactor). The
   `remote_runtime.py` site uses the weaker regex
   `redact_text_secrets` even though `self.config` is presumably a
   pydantic object. Either upgrade that site to `sanitize_config` or
   document why regex is acceptable there.

2. **No test changes visible.** Security fixes should ship with
   tests asserting `assert "secret-value" not in caplog.text`. Even
   one regression test per call site would be enough; otherwise the
   next refactor that drops the wrapper won't fail CI.

3. **`update_and_remount` in `proxy/manager.py:154-159`** *adds* a
   new info log. That's a behavior change beyond redaction —
   suddenly every proxy update produces a log line. Confirm log
   volume is acceptable in production (or downgrade to `debug`).

## Nits

- Worth a one-line CHANGELOG / SECURITY note: previously, MCP error
  surfaces (UI error collector + logs) could echo URL secrets verbatim.
- `_redact_compat` is the right module name (it already exists), so
  no surface-area concern.
