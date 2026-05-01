# Review: openai/codex #20508 — Add config-backed stdio exec-server environments

- URL: https://github.com/openai/codex/pull/20508
- Head SHA: `d4612c73c5883f9f0ff65d230a924e47f56e6ca9`
- Files: 21
- Size: +910/-101

## Summary of intended change

Three coupled additions to the exec-server environment subsystem:

1. **Stdio transport for exec-server clients** (`exec-server/src/client.rs`):
   adds `ExecServerTransport::StdioShellCommand(String)` alongside the
   existing `WebSocketUrl(String)`, plus
   `ExecServerClient::connect_stdio_command` at `:319-364` that spawns
   the configured shell command (via `sh -lc` on unix, `cmd /C` on
   Windows), pipes stdin/stdout into a `JsonRpcConnection::from_stdio`,
   forwards stderr to `tracing` debug, and registers a `StdioChildGuard`
   resource that kills the child on drop.

2. **`environments.toml` config provider** (`exec-server/src/environment_provider.rs`):
   new `TomlEnvironmentProvider` reading `$CODEX_HOME/environments.toml`
   with shape:
   ```toml
   default = "ssh-dev"   # or "none" to disable, omit to default to "local"
   [[items]]
   id = "devbox"
   url = "ws://127.0.0.1:8765"
   [[items]]
   id = "ssh-dev"
   command = "ssh dev \"codex exec-server --listen stdio\""
   ```
   `validate_environment_item` at `:1019-1058` enforces: id non-empty,
   no surrounding whitespace, not `local`/`none` (reserved), exactly one
   of `url` or `command`, ws/wss scheme for url. The provider's
   `default_environment_selection()` returns `Disabled` for `default = "none"`,
   otherwise the named id (with `LOCAL_ENVIRONMENT_ID` as the implicit
   fallback when `default` is omitted).

3. **`DefaultEnvironmentSelection` enum** plumbed through
   `EnvironmentManager::from_provider_environments` so the manager can
   distinguish three cases — `Derived` (legacy: prefer remote, fall back
   to local), `Environment(id)` (toml-named default with validation that
   id exists), `Disabled` (no default at all). Construction now returns
   `Result<Self, _>` instead of `Self`, with the unknown-default-id case
   bubbling as `ExecServerError::Protocol`.

Plus mechanical `EnvironmentManager::new(...).await?` propagation across
five call sites: `app-server/src/lib.rs:431-444`, `core/src/connectors.rs:200-208`,
`core/src/prompt_debug.rs:38-50`, `mcp-server/src/lib.rs`,
`thread-manager-sample/src/main.rs`, and the CLI exec-server `--listen`
help string at `cli/src/main.rs:449` extended with `stdio` / `stdio://`.

## Review

### Stdio transport mechanics

The `connect_stdio_command` implementation at `client.rs:319-364` is mostly
right but has three risk surfaces worth pinning:

- **`sh -lc`** at `:639-642` runs the shell command in a *login* shell.
  That sources the user's profile (`~/.profile`, `~/.bashrc` via
  `BASH_ENV`, etc.) which can introduce per-host startup output that
  pollutes the JSON-RPC stream. Recommend `sh -c` (no `-l`) unless there's
  a specific reason — exec-server commands like
  `ssh dev "codex exec-server --listen stdio"` do their own login on the
  remote, locally we just want to spawn the command. If `-l` is
  intentional (e.g. for `nvm`-style PATH munging), document it in the
  TOML schema doc.

- **Stderr handling** at `:340-355`: the spawned task forwards stderr
  line-by-line to `tracing::debug!`. Two follow-ups: (a) on `Err(_)` from
  `lines.next_line()` the loop `break`s but leaves the task to exit
  silently — fine, but a single `warn!` summarizing how many lines were
  read before the error would help diagnose protocol-frame-on-stderr
  shapes. (b) If the child spews several MB of stderr before crashing,
  every line allocates a `String` and goes to the global tracing
  subscriber; consider a per-spawn rate limit.

- **`StdioChildGuard::drop`** at `:614-627`: when no tokio runtime is
  available (`Handle::try_current()` errs), `terminate_stdio_child_now`
  is called which only invokes `child.start_kill()` and never `wait()`s.
  That leaves a zombie. In practice, `Drop` running outside a tokio
  context for a child that was spawned inside one is rare, but if it
  happens (e.g. async drop racing a runtime shutdown) the zombie is
  unbounded. Acceptable for now; flag a `// TODO: best-effort wait
  on a blocking thread` comment.

### TOML schema and validation

Excellent shape:
- `#[schemars(deny_unknown_fields)]` on both `EnvironmentsToml` and
  `EnvironmentToml` at `environment_provider.rs:907,915` means a typo'd
  key fails parse rather than being silently ignored.
- The `validate_environment_item` at `:1019-1058` covers the four
  url/command shapes correctly: `(Some, None) -> ws-validate`,
  `(None, Some) -> non-empty`, `(Some, Some) -> reject "must set
  exactly one"`, `(None, None) -> reject "must set exactly one"`.
- Reserved-id check at `:1037-1041` correctly excludes both
  `LOCAL_ENVIRONMENT_ID` (case-sensitive) and any case-variant of `none`
  (`eq_ignore_ascii_case`). That asymmetry is *intentional and right*:
  `"none"` is a `default` sentinel so any case variant must not collide
  with a real id, but `"local"` is a real id that just happens to be
  reserved.
- The `default = "none"` sentinel handling at `:953-960` and again at
  `:984-988` is duplicated — the validation in `TomlEnvironmentProvider::new`
  rejects an unknown default *unless* it's the case-insensitive `"none"`
  string, and `default_environment_selection()` separately returns
  `Disabled` for the same string. Fine, but worth pulling into a single
  `parse_default_selection(&Option<String>) -> ParsedDefault` helper so
  the case-sensitivity rules can't drift.

### Tests

Solid coverage:
- `connect_stdio_command_initializes_json_rpc_client` at `client.rs:1078-1090`
  exercises the real spawn path with a tiny shell-script protocol mock,
  pinning that `sessionId` round-trips. `#[cfg(not(windows))]` correctly
  scopes it to the unix shell.
- `environment_manager_uses_explicit_provider_default` /
  `_disables_provider_default` / `_rejects_unknown_provider_default` at
  `environment.rs:506-560` cover the three `DefaultEnvironmentSelection`
  arms.
- `toml_provider_adds_implicit_local_and_configured_environments` at
  `environment_provider.rs:386-422` round-trips a realistic two-env config
  through `get_environments` and confirms whitespace trimming.
- `toml_provider_rejects_invalid_items` (truncated in diff at line 1100+)
  appears to enumerate every reject path. Good.

What's missing: a test that confirms
`environment_provider_from_codex_home(path_with_no_toml)` falls back to
`DefaultEnvironmentProvider::from_env()` cleanly (i.e. the absence of
`environments.toml` is not an error). That's the "current behavior preserved"
contract for users who haven't opted into the new config and is worth
pinning.

### Cross-crate plumbing

Five call sites switched to `?` on `EnvironmentManager::new`. The signature
change is a hard break for any out-of-tree consumer of `codex-exec-server`,
which is acceptable here but should be called out in the PR body. The
`prompt_debug.rs:50` site uses `CodexErr::Fatal(err.to_string())` to bridge
into the codex-protocol error space, which loses the `ExecServerError`
variant — fine for a debug path, but the `app-server/src/lib.rs:444`
production path uses `std::io::Error::other` which is the right shape.

## Verdict

**merge-after-nits**

Wants:
- Drop `-l` from `sh -lc` at `client.rs:642` (or document why a login
  shell is required for exec-server stdio commands; pollution from
  user profile output on stderr is the most likely incident shape).
- Add a test that `environment_provider_from_codex_home` returns the
  `DefaultEnvironmentProvider` (not an error) when `environments.toml`
  is absent, to pin the "no config = no behavior change" contract.
- Pull the `default = "none"` case-insensitive sentinel into a single
  `parse_default_selection` helper so `TomlEnvironmentProvider::new`
  validation and `default_environment_selection()` can't drift.
- Optional: `// TODO: blocking-wait fallback` comment on
  `terminate_stdio_child_now` at `client.rs:619-621` for the
  no-tokio-runtime drop case.
- Optional: PR body callout that this is a minor breaking change for
  any out-of-tree consumer of `codex-exec-server::EnvironmentManager::new`
  (signature returns `Result` now).

The architecture — provider trait + named-default selection + transport
enum — is the right shape for what comes next (named environments, ssh
exec-server, future http transport).
