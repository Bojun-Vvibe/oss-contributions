# openai/codex PR #20147 — feat: add managed network proxy feature flag

- Repo: openai/codex
- PR: https://github.com/openai/codex/pull/20147
- Head SHA: `94f65a149e7b0130d93abca49adf23824212a320`
- Author: codex contributor
- Size: +479 / −11, 9 files
- Follow-up to: #19900

## Context

The permissions migration cited in #19900 made
`permissions.<profile>.network.enabled` the canonical sandbox-network bit.
The collateral problem: managed-proxy startup was being implicitly tied to
that same bit, which conflates two genuinely different concerns. "I want
the sandbox to allow direct outbound traffic" and "I want codex to start
the managed network proxy" are independent decisions — a user on a legacy
sandbox mode may want the proxy without enabling sandbox-direct, and a
user with `network.enabled = true` may not want the proxy at all. This
PR gives the managed proxy its own feature surface
(`features.network_proxy`) so the two settings don't have to ride the
same flag.

## What the diff actually does

The change has four moving parts:

1. **A new `Feature::NetworkProxy` enum variant + a configurable
   `NetworkProxyConfigToml`** added to the features registry. The TOML
   shape (visible in `config.schema.json`) is rich:
   `enabled`, `proxy_url`, `socks_url`, `mode`
   (`limited`/`full`), `enable_socks5`, `enable_socks5_udp`,
   `allow_local_binding`, `allow_upstream_proxy`,
   `dangerously_allow_non_loopback_proxy`,
   `dangerously_allow_all_unix_sockets`, plus `domains` and
   `unix_sockets` permission tables. That's a substantial config
   surface for an experimental feature.
2. **CLI feature-toggle plumbing reshaped to preserve config tables.**
   The `cli/src/main.rs:680-700` rewrite swaps
   `features.{name}=true|false` overrides for
   `features.{name}.enabled=true|false` whenever the feature has its
   own configurable table, so `--enable network_proxy` no longer
   replaces the entire `[features.network_proxy]` table with a bare
   boolean. The dispatch happens through the new
   `feature_toggle_override_key` helper, returning `Option<String>` to
   distinguish unknown features from configurable ones.
3. **Schema generation gets a special-case branch** at
   `config/src/schema.rs:34-44`: when iterating features for
   `features_schema`, `Feature::NetworkProxy` returns
   `FeatureToml<NetworkProxyConfigToml>` rather than the default
   `bool`. This mirrors the existing `MultiAgentV2` branch.
4. **`Config::load_from_base_config_with_overlay` overlays the feature
   table onto the configured proxy state after permission-profile
   selection** at `core/src/config/mod.rs:1935-1940` and
   `2182-2189`, so enabling `features.network_proxy` can start the
   managed proxy without changing the active `NetworkSandboxPolicy`.

The test matrix at `core/src/config/config_tests.rs:773+` covers four
scenarios:

- `network_proxy_feature_starts_proxy_without_enabling_sandbox_network`
  pins the headline contract — feature enables proxy startup,
  `network_sandbox_policy()` stays `Restricted`.
- (per PR body) merging CLI overrides into the feature config table
- preserving managed `[experimental_network]` startup without the new
  feature (back-compat for legacy users)
- reusing profile-supplied proxy settings when the feature is enabled

Plus the CLI-side test
`feature_toggles_preserve_configurable_feature_tables` at
`cli/src/main.rs:2545-2557` pinning that the new `.enabled=` shape is
emitted for both `network_proxy` (configurable) and `multi_agent_v2`
(also configurable, per existing precedent).

## What I'd want to see before merge

1. **Schema-config drift risk on `dangerously_*` fields.** Three
   fields in `NetworkProxyConfigToml` carry the `dangerously_` prefix
   (`dangerously_allow_non_loopback_proxy`,
   `dangerously_allow_all_unix_sockets`). For an experimental feature
   that gates proxy startup independently of sandbox network, a future
   contributor adding a fourth dangerous field should be forced through
   a checklist. A `// CHECKLIST: any new `dangerously_` field must:
   (a) default to false, (b) be excluded from auto-overlay from
   features.network_proxy when permission-profile sets a more
   restrictive value, (c) emit a startup-time warning` comment near
   the struct definition would help.

2. **Schema branch in `features_schema` is now two special cases.**
   `MultiAgentV2` and `NetworkProxy` each get a hand-written `if`
   branch. Three would tip this into needing a registry pattern —
   features carry a `schema_for(&self, gen) -> Schema` that defaults
   to `bool` and gets specialized per variant. Worth a TODO before the
   third configurable feature lands.

3. **The CLI tests pin override-key emission but not the round-trip.**
   `feature_toggles_preserve_configurable_feature_tables` asserts the
   string `"features.network_proxy.enabled=true"` is produced, but
   doesn't assert that *applying* that override to a config file
   actually preserves the rest of the `[features.network_proxy]`
   table. A small round-trip test (load a TOML with `proxy_url`
   set, apply `--enable network_proxy`, reload, assert `proxy_url`
   survived) would lock the actual contract.

4. **`feature_toggle_override_key` returns `Option<String>` and the
   CLI maps `None` to "Unknown feature flag".** That's correct, but it
   means typos like `--enable network-proxy` (hyphen instead of
   underscore) get the same error as a genuinely unknown feature. A
   small "did you mean: network_proxy?" suggestion using
   `strsim::normalized_levenshtein` against the known-feature list
   would catch the most common user error class.

5. **`mode = "limited" | "full"` semantics aren't visible in the
   schema's description.** A reviewer reading `config.schema.json`
   sees the enum but no docs on what each mode permits. JSON-Schema
   `description` fields on the enum (or at minimum a rustdoc on
   `NetworkProxyModeToml` that schemars picks up) would make the
   generated docs self-explanatory.

6. **Back-compat with `[experimental_network]` is preserved per the
   third test case, which is the right call.** Worth a CHANGELOG line
   ("`[experimental_network]` continues to work as-is; new code should
   use `[features.network_proxy]`").

## Suggestions

- Add the `dangerously_*` checklist comment near
  `NetworkProxyConfigToml`.
- TODO for refactoring `features_schema` into a registry once a third
  configurable feature appears.
- Round-trip test that applies the new `.enabled=` override and
  asserts the rest of the table survives.
- Levenshtein "did you mean" suggestion for unknown-feature CLI errors.
- JSON-Schema `description` on `NetworkProxyModeToml` enum variants.

## Verdict

**merge-after-nits** — clean separation of concerns, well-tested at the
unit level, and the back-compat story for `[experimental_network]` is
intact. The nits are about hardening the dangerously-prefixed fields
and locking in the round-trip behavior of the new override shape.
