# block/goose PR #8893 — refactor: use Config::global() instead of per-instance config in SACP server

- Repo: block/goose
- PR: https://github.com/block/goose/pull/8893
- Head SHA: `70382ad7b081b4861115942a6266c974a8ba5134`
- Author: matt2e
- Size: +104 / −110, 3 files

## Context

`GooseAcpAgent` was holding a `config_dir: PathBuf` field and re-reading
the YAML config from disk on every operation that needed it
(`load_config()`, `config()`, `resolve_provider_and_model`,
`on_new_session` background task setup). Every call rebuilt
`Config::new(config_dir.join(CONFIG_YAML_NAME), "goose")` — meaning every
session-init path paid the file-IO + parse cost, and any change to
`config.yaml` was picked up the next time anyone called these helpers
(unintended live-reload semantics in places that didn't ask for it).

The PR migrates all of these to `Config::global()`, which returns a process-wide
`OnceCell`-backed config singleton initialized once at boot. Net: -110 lines
of repeated `Config::new` plumbing, -1 field, -2 helpers, the per-session
`load_config()` `match` ladder collapses by one indent level, and config
read semantics become "snapshot at boot, no live-reload" — which is what
every other goose subsystem already assumed about provider config.

## What the diff actually does

1. `crates/goose/src/acp/server.rs:6-9` — drops
   `use crate::config::base::CONFIG_YAML_NAME;` (no longer referenced).
2. `:184-194` — deletes `config_dir: std::path::PathBuf` field from
   `GooseAcpAgent`.
3. `:721-731` — `resolve_provider_and_model` loses its `config_dir`
   parameter; body becomes a one-line forward to
   `resolve_provider_and_model_from_config(Config::global(),
   goose_session)`.
4. `:843-869` — constructor stops storing `config_dir` (it's still
   threaded into `PermissionManager::new(config_dir)` at `:847`, which
   needs the path for permission file lookups, then dropped on the
   floor).
5. `:865-869` — deletes `load_config()` and `config()` helper methods
   entirely.
6. `:900-1003` — the giant `if should_refresh_inventory_for_session_init`
   branch loses its outermost `match self.load_config()` arm. The body
   shifts left by one indent level, with `config = Config::global()`
   substituted at the top. The previous `Err(error) => warn!(...,
   "failed to load config during synchronous inventory refresh")` arm
   is deleted (since `Config::global()` is infallible).
7. `:1021-1042` — same pattern in the background `phase1` task: drops
   the `match Config::new(...)` ladder for a single
   `let config = Config::global();`. The `error!(error = %msg,
   "Background agent setup failed (config)")` early-return is gone.
8. `:1042-1100` — `&config` becomes `config` (Config::global() returns
   `&'static Config`), threading through
   `resolve_provider_and_model_from_config(config, ...)` and
   `EnabledExtensionsState::extensions_or_default(Some(...), config)`.

## Risks and gaps

1. **Loss of the `load_config()` failure path is the one real
   behavioral change**. Previously, if `config.yaml` was malformed at
   the moment of the request, the per-session error would surface in
   the spawn channel as `agent_tx.send(Some(Err(msg)))` with the
   parser error. Now `Config::global()` is initialized once at boot
   and always returns the cached parse — so a corrupted config gets
   detected at startup (and presumably crashes the proxy on bring-up),
   not on the first session attempt. That's a strictly better failure
   mode for production deployments (fail fast at boot), but for local
   dev where `config.yaml` is being edited live, the old behavior at
   least surfaced a recoverable error per-session. Worth a CHANGELOG
   entry that "config errors now surface at startup; live edits to
   config.yaml require a goose restart to take effect."

2. **`Config::global()`'s init point is invisible from this diff**.
   Verify there is exactly one call site that initializes the global
   (likely in `goose-cli`'s `main` or in `goose-server`'s bootstrap),
   and that it runs before any `GooseAcpAgent::new` ever calls
   `Config::global()` for the first time. If `Config::global()`
   silently lazy-inits on first read with default values when uninitialized,
   any test or alternative entrypoint that constructs `GooseAcpAgent`
   directly without going through the bootstrap will get an empty config
   and silently misbehave.

3. **`config_dir` is still passed into `PermissionManager::new(config_dir)`
   at line 847**. After this PR, the `config_dir` parameter to
   `GooseAcpAgent::new` is consumed *only* by `PermissionManager`, never
   stored, never read again. Worth documenting that the field's purpose
   is now "where to look for permission files, distinct from the global
   config snapshot" — otherwise a future contributor will reasonably
   ask "why does the constructor take config_dir if we're using
   `Config::global()`?".

4. **Live-reload regression for any field consumed downstream**. The
   `EnabledExtensionsState::extensions_or_default(Some(&goose_session.extension_data),
   config)` call at `:912` previously read fresh-from-disk extension
   config per session-init. Now it reads the boot-time snapshot. If a
   user enables a new extension via `~/.config/goose/config.yaml` while
   the server is running, the extension list is now frozen until restart
   — previously the next `on_new_session` would have picked it up. If
   that's a UX regression, the fix is either (a) document the new
   no-live-reload contract, or (b) add a `Config::reload()` API that
   the server invokes on SIGHUP / explicit reload command.

5. **No tests added**. This is a behavioral change (config caching
   semantics) and it lands without a test. A test that constructs
   `GooseAcpAgent`, invokes `Config::global()` mock-overridable, and
   asserts the same provider/model is resolved across two consecutive
   `on_new_session` calls would lock the new contract. The existing
   ACP server test suite presumably catches regressions on the
   call-site signatures, but not on the snapshot semantics.

6. **The constructor no longer stores `config_dir`, but the parameter
   stays**. Either drop the parameter entirely (let the caller pass
   `PermissionManager` already constructed) or rename to
   `permission_dir` to reflect its actual scope post-refactor. As-is,
   the parameter name lies about what it controls.

7. **The `phase1` background task's deleted `Err` arm** had a
   `let _ = agent_tx.send(Some(Err(msg)));` that propagated the
   config-load error to the channel reader. After this PR there's no
   plausible failure point at that step, so the channel reader's
   error-handling code for config-load failures is now dead. Worth
   greppinging for `agent_tx` consumers and confirming nothing
   semantically depended on receiving a config-load `Err`.

## Suggestions

- Add a CHANGELOG entry covering the live-reload regression: config
  changes now require a server restart.
- Verify and document the `Config::global()` init invariant: it must
  be initialized at server bootstrap before any `GooseAcpAgent::new` is
  called.
- Drop the `config_dir` parameter from `GooseAcpAgent::new` in favor of
  taking a constructed `PermissionManager`, OR rename it `permission_dir`
  for honesty.
- Add a unit test that asserts `on_new_session` resolves provider/model
  consistently across calls (snapshot semantics).
- Optionally, add a `Config::reload()` for SIGHUP-style live-reload, if
  the UX regression matters.
- Audit `agent_tx` receivers for now-dead config-load-error handling.

## Verdict

**merge-after-nits** — the refactor is correct and the simplification is
real (-110 deduplicated lines, one indent level off the inventory-refresh
ladder). The live-reload behavioral change is the single substantive shift
and deserves a release-note callout, but is otherwise a strictly-better
production failure mode (fail at boot, not per-session).
