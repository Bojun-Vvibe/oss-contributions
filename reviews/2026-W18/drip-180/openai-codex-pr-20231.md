# openai/codex#20231 — [apps] Add apps MCP path override

- **PR:** https://github.com/openai/codex/pull/20231
- **Author:** @adaley-openai
- **Head SHA:** `3e1b5287` (full: `3e1b52874cde7f0cc56b3fb326bd0e165f268e99`)
- **Size:** +104/-11 across 6 files

## What this does

Adds a new optional `apps_mcp_path` config key (TOML root-level) so the path
suffix used for the built-in apps MCP server URL can be overridden.

The threading is straightforward and complete:

1. `codex-rs/config/src/config_toml.rs` — adds
   `pub apps_mcp_path: Option<String>` to `ConfigToml`.
2. `codex-rs/core/src/config/mod.rs:642` — adds
   `pub apps_mcp_path: Option<String>` to the `Config` struct, plus the
   plumb through `Config::load_from_base_config_with_overrides`
   (`apps_mcp_path: cfg.apps_mcp_path,` at the new line in the
   `Config { ... }` constructor) and through `to_mcp_config` so it lands on
   `McpConfig` (`codex-mcp/src/mcp/mod.rs:96`).
3. `codex-rs/codex-mcp/src/mcp/mod.rs:380` — `codex_apps_mcp_url_for_base_url`
   gains a second `apps_mcp_path_override: Option<&str>` param. The function
   now picks `(base_url, default_path)` per the existing `/backend-api` /
   `/api/codex` / "neither" branches and then layers the override on top
   with `apps_mcp_path_override.unwrap_or(default_path).trim_start_matches('/')`.
4. `core/config.schema.json` — adds the corresponding JSON schema entry.

## Design note that's actually nice

The default-vs-override split was done as a tuple `(base_url, default_path)`
so the override only replaces the path component, never the base. That
avoids the trap where an override like `"/custom/mcp"` against
`http://localhost:8080` (no path) would otherwise lose the implicit
`/api/codex` base. The new test
`codex_apps_mcp_url_for_base_url_uses_configured_path` verifies all four
base-URL shapes resolve to `…/custom/mcp` correctly, including the
no-base-path case at `mod_tests.rs:142`:

```
codex_apps_mcp_url_for_base_url("http://localhost:8080", Some("/custom/mcp"))
  == "http://localhost:8080/api/codex/custom/mcp"
```

## Concerns

1. **No validation on the override.** The override is taken verbatim,
   `trim_start_matches('/')`'d, and concatenated. So a user can pass
   `apps_mcp_path = "../escape"` or `apps_mcp_path = "https://evil/"` and
   produce an unexpected URL — though since the base is hard-prefixed, the
   second case still ends up as `…/https://evil/`, which is malformed but
   not a security primitive. Worth at minimum a `debug_assert!` or a doc
   comment on `apps_mcp_path` saying "single relative path segment(s),
   no scheme, no `..`".

2. **Schema description is thin.** `core/config.schema.json` adds:

   ```
   "apps_mcp_path": {
     "description": "Path override for the built-in apps MCP server.",
     "type": "string"
   }
   ```

   That doesn't tell the user it must be a path component (no scheme, no
   leading `..`). Compare with the more verbose descriptions for other root
   keys in the same file.

3. **Config-precedence test coverage exists, but only one shape.** The new
   `config_loads_apps_mcp_path_from_toml` test
   (`config/config_tests.rs:7090`) covers the load path with one example.
   It would be slightly stronger to also assert that an override at a
   profile level (if profiles even support this — does
   `ConfigProfile` propagate it?) behaves consistently. The diff only
   touches the `ConfigToml` root struct, so I'd guess profile overrides
   silently won't apply. If that's intentional, document it.

4. **`apps_mcp_path` vs `apps.mcp_path` namespacing.** Codex already has
   an `apps` config block (the diff mentions `config.apps_enabled`). It's
   slightly weird to put this under the root rather than nested under
   `apps`, especially since the field is "the path the apps MCP serves
   from". A future reader will probably grep `apps.` and miss this.

## Verdict

**`merge-after-nits`** — clean, well-tested, narrow surface. The placement
under root rather than `apps.mcp_path`, plus the missing input validation,
are worth a one-line clarification before merge.

## Nits

- Tighten the doc comment / schema description with the constraints on the
  override value.
- Consider moving the key under the `apps` config section for namespace
  hygiene (`apps.mcp_path`) so it sits next to `apps_enabled` and friends.
