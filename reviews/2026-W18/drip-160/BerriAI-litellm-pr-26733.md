# BerriAI/litellm PR #26733 — feat(mcp): opt-in short-ID tool prefix to keep MCP tool names under the 60-char limit

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26733
- Head SHA: `fc49c181bca73d63d3b861d7fc2a46a834a3ba8e`
- State: OPEN, +362/-40 across 4 files

## What it does

Adds an opt-in `LITELLM_USE_SHORT_MCP_TOOL_PREFIX` env-flag that swaps the historical alias/server_name prefix on MCP tool/prompt/resource/resource-template names for a deterministic three-character base62 prefix derived from `SHA-256(server_id)[:8] → base62 → first 3 chars`. Motivated by some model APIs (Anthropic etc.) capping `tools[].function.name` at 60 chars — a long upstream tool name + a long server name routinely overflowed. To keep mixed-version clients working through rollout, every reverse-lookup site is rewritten to consult a new `iter_known_server_prefixes(server)` iterator that yields *all* known forms (current prefix, alias, server_name, server_id, short-ID) and registers them all into `tool_name_to_mcp_server_name_mapping`.

## Specific reads

- `utils.py:178-184` — the deterministic prefix derivation:
  ```python
  def compute_short_server_prefix(server_id: str) -> str:
      if not server_id:
          raise ValueError("compute_short_server_prefix requires a non-empty server_id")
      digest = hashlib.sha256(server_id.encode("utf-8")).digest()
      value = int.from_bytes(digest[:8], "big")
      chars = []
      for _ in range(SHORT_MCP_TOOL_PREFIX_LENGTH):
          value, idx = divmod(value, len(_BASE62_ALPHABET))
          chars.append(_BASE62_ALPHABET[idx])
      return "".join(reversed(chars))
  ```
  62³ = 238 328 — the design comment correctly acknowledges collision is possible but argues it only affects the cosmetic emitted name, not routing correctness, because the reverse-lookup map registers *all* prefix forms. Worth a follow-up: if two registered servers happen to hash-collide to the same three-char prefix, the **emitted** tool name `<short>-<tool>` is ambiguous on the wire — the model sees two functionally identical names and downstream routing would non-deterministically pick whichever server registered first via `setdefault` at `mcp_server_manager.py:2615`. A startup-time collision check (warn or refuse) is missing.

- `mcp_server_manager.py:2613-2617` — the new `prefix_to_server` lookup:
  ```python
  prefix_to_server: Dict[str, MCPServer] = {}
  for server in registry_servers:
      for known_prefix in iter_known_server_prefixes(server):
          normalised = normalize_server_name(known_prefix)
          prefix_to_server.setdefault(normalised, server)
  ```
  `setdefault` is the right choice for "first-write-wins on collision" but combined with the lack of collision warning at registration time means a silent routing-stickiness bug on prefix collision. Also, the dict is rebuilt on **every** `_get_mcp_server_from_tool_name` call — for high-QPS proxies with many registered servers this is noticeable allocation pressure on a hot path. A cache invalidated on registry change would be a structural improvement.

- `mcp_server_manager.py:1840-1845` — the registration site:
  ```python
  self.tool_name_to_mcp_server_name_mapping[original_name] = prefix
  for known_prefix in iter_known_server_prefixes(server):
      qualified = add_server_prefix_to_name(original_name, known_prefix)
      self.tool_name_to_mcp_server_name_mapping[qualified] = prefix
  ```
  The intent is clear: register every form so old clients still resolve. But this means a tool registered when `LITELLM_USE_SHORT_MCP_TOOL_PREFIX=true` will populate the map with both `<short>-<tool>` AND `<long>-<tool>`. If the operator later flips the flag off and a tool happens to have a name that collides with another server's `<long>-<tool>`, the order of registration determines which server wins — silent and non-obvious.

- `server.py:2034-2039` — the call_tool resolution rewrite:
  ```python
  # Resolve the actual MCP server up-front so the permission check uses
  # the canonical server.name even when the tool name is prefixed with a
  # short ID (LITELLM_USE_SHORT_MCP_TOOL_PREFIX) that doesn't match the
  # server's display name directly.
  mcp_server = global_mcp_server_manager._get_mcp_server_from_tool_name(name)
  if mcp_server is not None:
      server_name = mcp_server.name
  ```
  This is a behavioral change beyond the stated scope: previously the `_get_mcp_server_from_tool_name` call only fired when `not server_name` (unprefixed tool names); now it *always* fires, and unconditionally overwrites `server_name` from `split_server_prefix_from_name`. For prefixed tool names where the resolution disagrees with the parsed prefix (e.g. an alias rename mid-session), the behavior silently flips. That's likely the right call but the comment doesn't acknowledge the broader scope.

- Test surface (`test_short_mcp_tool_prefix.py`) is solid — 207 lines covering the determinism contract, env-flag truthy/falsey parsing matrix, fallback-when-no-server-id, mixed-mode resolve in both directions (long-prefix-in-short-mode and vice versa), and a `<60` length pin at line 504. Notably **missing**: a collision test that constructs two `server_id` values whose SHA-256 prefix happens to fold to the same three base62 chars, then asserts whether the system warns/raises/silently picks-first. Hard to construct deterministically but worth a brute-force generator in the test fixture.

## Verdict: `merge-after-nits`

## Rationale

The motivation is real (60-char limit is a recurring footgun), the migration story is well-thought-out (every prefix form registered for reverse-lookup, opt-in flag, deterministic-without-persistence prefix derivation), and the test matrix exercises the right invariants. Three nits to address before merge: (a) add a startup-time collision check across all registered servers' short prefixes — a warn-at-startup or even refuse-to-start on collision is cheap and prevents the silent "first-registered wins" routing-stickiness bug the docstring hand-waves at; (b) cache `prefix_to_server` and invalidate on registry mutation rather than rebuilding on every `_get_mcp_server_from_tool_name` call; and (c) the `server.py:2034-2039` change widens the resolution scope from "unprefixed only" to "always, overwriting parsed prefix" — that's likely correct but should be called out in the docstring as a behavioral change in its own right, not a side effect of the short-prefix feature.
