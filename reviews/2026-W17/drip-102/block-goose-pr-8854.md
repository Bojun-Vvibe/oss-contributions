# block/goose PR #8854 — Lifei/remove defaults yaml

- Link: https://github.com/block/goose/pull/8854
- Head SHA: `7c60c1b3e8087d79c48e81fa1c338166557b7433`
- Size: +28 / -140 across 3 files

## Summary

Removes the `defaults.yaml` config-overlay mechanism from `crates/goose/src/config/base.rs` (the `defaults_path` field, `Config::new_with_defaults` constructor, `find_workspace_or_exe_root()` probe, `load_defaults` / `merge_missing_defaults` helpers, and four corresponding tests) and replaces the only known consumer — security defaults — with a one-shot `set_security_defaults()` call wired into `crates/goose-server/src/commands/agent.rs::run()` and `SecurityManager::new()`. The replacement reads `DEFAULT_SECURITY_PROMPT_ENABLED` / `DEFAULT_SECURITY_COMMAND_CLASSIFIER_ENABLED` env vars and seeds the corresponding `SECURITY_*` config keys only when they aren't already set.

## Specific-line citations

- `crates/goose/src/config/base.rs:122-167` removes the `defaults_path: Option<PathBuf>` field from `Config`, the `find_workspace_or_exe_root()` probe in `Default::default()`, and the `defaults_path: defaults_path` assignment in the constructor — all three layers of the overlay are now gone.
- `:284-310` removes `Config::new_with_defaults<P1, P2, P3>(config_path, secrets_path, defaults_path)` constructor entirely. **This is a breaking API change** for any external crate (or downstream fork) that called this constructor.
- `:339-343` simplifies `Config::load()` from `let mut values = self.load_raw()?; self.merge_missing_defaults(&mut values); Ok(values)` to just `self.load_raw()` — defaults are no longer merged at read time.
- `:705-721` deletes `load_defaults` (reads + parses `defaults.yaml`) and `merge_missing_defaults` (inserts default keys not present in user config) — these were the actual overlay logic.
- `:1838-1939` deletes four tests: `test_defaults_fallback_when_key_not_in_config`, `test_full_precedence_env_over_config_over_defaults`, `test_no_defaults_file_behaves_as_before`, `test_defaults_not_persisted_on_write`. The last one in particular pinned the contract that defaults never got written into user config — that contract no longer needs pinning because there are no defaults, but the *new* contract ("env-var defaults seeded into user config on first launch") is **not pinned** by any new test.
- `crates/goose/src/security/mod.rs:16-37` is the replacement: `set_security_defaults()` calls `set_default_if_not_exist(config, key, default_env)` for the two security keys. The helper checks `config.get_param::<bool>(key).is_ok()` to decide skip-vs-seed and then `config.set_param(key, parsed)` writes through to user config. Wired in two places: `goose-server/src/commands/agent.rs:40` (early in `run()`) and `SecurityManager::new()` (`security/mod.rs:55`).

## Verdict

**request-changes**

## Rationale

Five concrete blockers — none about the goal of removing dead overlay code (which is fine), but about how this specific PR executes it:

1. **PR description is empty.** Title is `Lifei/remove defaults yaml`, the Summary/Testing/Related Issues sections of the template are all literal placeholder comments. There is no statement of why `defaults.yaml` is going away, who else might be relying on it, what migration path users have if they were depending on it, or what "Lifei" means as a scope. For a 140-line deletion that removes a public constructor + a documented config-precedence layer, this is unacceptable in a project that takes config-precedence seriously enough to have written `test_full_precedence_env_over_config_over_defaults` in the first place.

2. **Public API removal with no deprecation.** `Config::new_with_defaults` is a public constructor on a public `Config` struct. Deleting it in one PR (rather than `#[deprecated]` for one release, then remove) breaks any downstream Goose embedder. A project at Goose's surface area should not silently break this without at minimum an entry in CHANGELOG / docs.

3. **Behavior change is not test-pinned.** The four deleted tests pinned the *old* "defaults overlay never persists to user config" contract. The new `set_security_defaults` does the **opposite** — it writes seeded values into user config via `set_param`. There is no test for the new contract: "env var present → first launch seeds key; subsequent launches don't reseed; user override of seeded key is preserved." Without that test, the next refactor of `set_default_if_not_exist` can silently flip the precedence and nothing catches it.

4. **`config.get_param::<bool>(key).is_ok()` conflates "key absent" with "key present but wrong type".** If `SECURITY_PROMPT_ENABLED` is somehow set to a string `"yes"` in a user's config, `get_param::<bool>` returns Err, and the helper happily reseeds with the env-var value, silently overwriting the user's misconfigured-but-intentional value. The check should be specifically against `ConfigError::NotFound`.

5. **Replacing a config-file overlay with env-var-driven seeding** changes the deployment model in ways the PR doesn't acknowledge. Previously: pkg ships `defaults.yaml`, users see "overrides apply on top of these". Now: pkg ships nothing, but `goose-server` writes env-var values into `~/.config/goose/config.yaml` on first run. This makes `config.yaml` no longer purely user-authored, which surprises operators who diff it for change tracking. Worth a paragraph in docs at minimum.

If the maintainers want to land the deletion, my request would be: (a) write a real PR description naming what was using `defaults.yaml` and why removal is safe, (b) `#[deprecated]` `new_with_defaults` for one release before removing, (c) add a test pinning that `set_security_defaults` is idempotent + preserves user overrides + doesn't reseed wrong-type keys, (d) tighten the `is_ok()` check to specifically `NotFound`.
