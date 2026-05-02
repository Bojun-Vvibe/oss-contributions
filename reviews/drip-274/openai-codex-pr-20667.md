# Review: openai/codex #20667 — Load configured environments from CODEX_HOME

- Repo: openai/codex
- PR: #20667
- Head SHA: `57e88850875dbc5477909c2229cef14af02a6a72`
- Author: starr-openai
- Size: +62 / -31 across 8 files

## What it does
Threads `codex_home` into `EnvironmentManagerArgs::new`, switches
`EnvironmentManager::new` to read environment definitions via
`environment_provider_from_codex_home` (TOML in `$CODEX_HOME`) instead of
reading only the `CODEX_EXEC_SERVER_URL` env var, and changes
`EnvironmentManager::new` to return `Result<Self, ExecServerError>` so TOML
parse failures surface as real errors. Updates 7 callers accordingly.

## File-level notes

**`codex-rs/exec-server/src/environment.rs` (head `57e8885`)**
- `EnvironmentManagerArgs` now carries `codex_home: PathBuf`. The
  `new(codex_home: impl AsRef<Path>, ...)` signature is ergonomic.
- `EnvironmentManager::new` doc comment correctly updated from
  "from `CODEX_EXEC_SERVER_URL`" to "from `CODEX_HOME`". Good — the old
  comment would now be misleading.
- The internal `from_default_provider_url` path still exists for the
  `from_exec_server_env` constructor; that preserves the env var as an
  alternative entrypoint. Make sure precedence between `CODEX_HOME` TOML and
  `CODEX_EXEC_SERVER_URL` is documented somewhere — readers may legitimately
  set both.

**`codex-rs/app-server/src/lib.rs` @ L420–446**
- `EnvironmentManager::new` construction was moved from before `find_codex_home()`
  to after, then wrapped with `.map_err(std::io::Error::other)?`. Correct
  ordering — `codex_home` is required before the manager can read TOML.

**`codex-rs/core/src/prompt_debug.rs` @ L40+**
- Wraps the new `Result` with `CodexErr::Fatal(err.to_string())`. `Fatal`
  is appropriate here because a malformed `environments.toml` should not
  silently degrade to an empty environment list. Consider attaching the
  source via `source` chaining instead of `to_string()` in a follow-up so
  the underlying parse error is visible in logs.

**`codex-rs/core/src/connectors.rs` @ L200–210**
- `.await?` propagation looks right. The function already returned a
  `Result`, so no signature churn.

**`codex-rs/exec/src/lib.rs`, `mcp-server/src/lib.rs`, `tui/src/lib.rs`,
`thread-manager-sample/src/main.rs`** — call-site mechanical updates. From
the diff hunks shown they look consistent.

## Risks
- Behavior change: a malformed `~/.codex/environments.toml` will now fail
  startup of the app-server / mcp-server / tui (was silently ignored).
  This is the right call but is user-visible — release notes should call
  it out.
- No new tests exercise the TOML loading path from these entrypoints. A
  smoke test that constructs `EnvironmentManager::new` against a temp
  `codex_home` with a malformed TOML and asserts the error type would
  protect the new error path.

## Verdict: `merge-after-nits`
Solid refactor with consistent call-site updates. Add a smoke test for
the new error propagation and document the precedence vs. the env var.
