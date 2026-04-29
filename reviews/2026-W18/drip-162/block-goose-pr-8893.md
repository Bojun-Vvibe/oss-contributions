# block/goose PR #8893 — refactor: use Config::global() instead of per-instance config in SACP server

- Repo: `block/goose`
- PR: https://github.com/block/goose/pull/8893
- Head SHA: `70382ad7b0`
- State: OPEN, +104/-110 across 3 files

## What it does

Migrates `GooseAcpAgent` (the SACP / Server-side Agent Client Protocol agent) from per-instance config-loading (`Config::new(config_dir.join(CONFIG_YAML_NAME), "goose")`) to the process-wide `Config::global()` singleton. PR description: "matches how the older REST APIs handled concurrent requests regarding config." Net diff is roughly even (-110/+104) because the change replaces explicit-arg-passing with global-fetch at each consumer site, eliminating both the `config_dir: PathBuf` field on `GooseAcpAgent` and the `load_config()`/`config()` helper methods.

Three files touched:

1. `crates/goose/src/acp/server.rs` — bulk of the change. Replaces every `self.load_config()` / `Config::new(self.config_dir.join(...), ...)` call with `Config::global()`. Drops the `config_dir` field from the struct (`:186`). Drops the `resolve_provider_and_model(config_dir, session)` helper's first arg (`:725-731`). Updates two call sites in `set_session_model` and another method to use `Config::global()` directly.
2. `crates/goose/src/acp/server_factory.rs` — likely just removes the `config_dir` argument from the `GooseAcpAgent::new(...)` call, but didn't read.
3. `crates/goose/src/config/base.rs` — likely a small change to `Config::global()` itself or to remove `CONFIG_YAML_NAME` from the public re-exports if no other consumer needs it.

## Specific reads

- `acp/server.rs:9` — drops `use ...config::base::CONFIG_YAML_NAME` import. Net: the constant is no longer referenced in this file. If it was only re-exported here, `base.rs` may need to drop the `pub` qualifier; if it's used elsewhere, this is just a local cleanup.
- `acp/server.rs:186` — `config_dir: std::path::PathBuf` field deleted from `GooseAcpAgent`. Reduces struct size by 24 bytes (PathBuf = pointer + len + cap), more importantly eliminates an entire class of "which config did this method see?" bugs in the multi-method case.
- `acp/server.rs:724-731` — `resolve_provider_and_model` no longer takes `config_dir`. Body collapses from `let config = Config::new(config_dir.join(CONFIG_YAML_NAME), "goose").map_err(|e| e.to_string())?;` to a single `resolve_provider_and_model_from_config(Config::global(), goose_session).await`. The function is now a thin wrapper around the `_from_config` variant — could arguably be deleted in favor of having the two callers call `_from_config(Config::global(), ...)` directly. The wrapper remains because it preserves the call-site shape (`resolve_provider_and_model(&goose_session)`) and lets future migration to a different config source happen in one place.
- `acp/server.rs:843-849` — constructor signature simplification. `permission_manager` is now built from `config_dir` (still passed in for that single use) but the field is no longer stored. Subtle: this means `PermissionManager` *captured* the `config_dir` at construction time and uses it for the lifetime of the agent — that's *different* from `Config::global()`, which can theoretically be reloaded. If `Config::global()` ever picks up a new `config_dir` (it can't today, but the abstraction admits it), the permission manager would still be reading the old one. Worth a comment.
- `acp/server.rs:855-858` — struct constructor literal drops `config_dir,` from the `Self { ... }` block. Compiler-enforced consistency with the field deletion.
- `acp/server.rs:868-877` — both `fn load_config(&self) -> Result<Config>` and `fn config(&self) -> Result<Config, sacp::Error>` are deleted. The two methods were thin wrappers that loaded from disk on every call (per Pre-PR; that's the perf claim in the description "eliminating redundant config file reads"). Their removal is the load-bearing perf win.
- `acp/server.rs:903-980` — the largest change site, in the session-init refresh flow. Pre-PR: `match self.load_config() { Ok(config) => { ... } Err(error) => warn!(... "failed to load config during synchronous inventory refresh") }`. Post-PR: drops the entire `Err` arm because `Config::global()` returns `&'static Config` (or equivalent) infallibly. Code is flatter and the never-taken error log goes away. Good.
- `acp/server.rs:1023-1031` — in the background `agent_setup` task, replaces an explicit `Config::new(...)` (which had its own error-arm sending failure back via `agent_tx`) with `Config::global()`. This **deletes the failure path entirely** — pre-PR a transient FS error reading `config.yaml` would surface as a setup failure; post-PR it can't because the global is already loaded. That's the right call for a singleton-loaded-at-startup pattern, but does mean an operator who *edits* `config.yaml` mid-process won't see changes until restart. Pre-PR they wouldn't either (the per-instance load was per-`GooseAcpAgent`-construction, not per-call), so net behavior is the same.
- `acp/server.rs:1045-1097` — multiple `&config` references become `config` (since `Config::global()` returns a reference, not an owned value). The argument-type changes are compiler-enforced consistent. `EnabledExtensionsState::extensions_or_default(Some(&...), config)` and `get_enabled_extensions_with_config(config)` and `EnabledExtensionsState::for_session(..., config)` all take `&Config` so the unprefixed `config: &Config` works.
- `acp/server.rs:1762, 2164` — the two `resolve_provider_and_model(&self.config_dir, &goose_session).await` call sites become `resolve_provider_and_model(&goose_session).await`. Mechanical.
- `acp/server.rs:2387, 2521` — two methods (`set_session_model`, the other one near `:2521`) replace `let config = self.config()?` with `let config = Config::global();`. The `?` propagation is now unnecessary because `Config::global()` is infallible. Methods could potentially have their `Result` return type tightened if no other `?` remains, but that's out of scope.

## Risk

1. **Hot-reload semantics removed silently**. Pre-PR, every method call re-read `config.yaml` from disk (slow but allowed mid-process edits to take effect). Post-PR all reads come from a process-loaded singleton. If any operator workflow depended on "edit config.yaml, signal goosed, see new value next call", that's gone. Per the PR description ("matches how the older REST APIs handled concurrent requests"), the REST path already had this property, so this is a parity fix not a regression — but a release-note bullet is warranted.
2. **`Config::global()` initialization order**. The PR assumes `Config::global()` is initialized before `GooseAcpAgent::new()` is ever called. If any test or a non-standard binary entry point constructs the agent without first initializing the global, this will panic or return a default-constructed config. Should add a debug-assert or explicit initialization-check at agent construction.
3. **Test isolation**: tests that previously could pass per-instance configs to construct distinct `GooseAcpAgent`s with distinct configs now have to mutate the global. Goose's test suite likely has some pattern for this (test-only `Config::set_global` or a `with_config` scope), but PRs that switch from per-instance to global config tend to expose latent test-isolation bugs. Worth a CI-run check to make sure no tests fail in parallel that pass serially.
4. **`config_dir` parameter still flows through `GooseAcpAgent::new(...)`** for the `PermissionManager` construction. The agent doesn't store it, but accepts it — leaving a minor signature asymmetry where `config_dir` is now a one-shot arg used only to seed one downstream. Could either (a) push `Config::global()`-derived `config_dir` into `PermissionManager::new()` directly and drop the arg, or (b) leave as-is and add a comment.

## Verdict

`merge-after-nits` — correct simplification that aligns ACP with REST semantics and removes a real perf footgun (per-call YAML disk read). Three nits to consider pre-merge: (a) release-note the loss of mid-process config reload, (b) add an init-order sanity check (`Config::is_global_initialized()` or similar) at the top of `GooseAcpAgent::new`, (c) decide whether `config_dir` should still be a constructor arg now that it only seeds `PermissionManager`.

## What I learned

The "per-instance config field that loads from disk on every read" anti-pattern is a slow-burn perf bug that hides in plain sight in any long-running async server — every method call quietly re-parses YAML, the cost is amortized into 100ms response times that look "normal," and the only way to find it is to profile or to do a refactor like this one. The right shape is process-wide singleton with explicit reload signal (SIGHUP handler that calls `Config::reload_global()`) for operators who actually need hot-reload. This PR doesn't add the reload signal — that's fine as a follow-up, but worth tracking.
