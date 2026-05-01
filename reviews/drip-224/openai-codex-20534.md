# openai/codex #20534 — Add graceful drain for exec-server shutdown

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20534
- **HEAD SHA:** `f6b3eec2deef7509612ff09370fea904503bf7a8`
- **Author:** starr-openai
- **Verdict:** `merge-after-nits`

## What the diff does

Multi-file `codex-exec-server` shutdown overhaul:

1. **New config surface.** `codex-rs/exec-server/src/config.rs` (new file)
   adds `ExecServerConfig` with a single `graceful_shutdown_timeout_ms:
   Option<u64>` field, `#[serde(deny_unknown_fields)]`,
   `EXEC_SERVER_CONFIG_FILE = "exec-server.toml"`, and
   `DEFAULT_GRACEFUL_SHUTDOWN_TIMEOUT = Duration::from_secs(30)`. Errors
   are typed via `ExecServerConfigError` (Read/Parse/Invalid).

2. **CLI plumbing.** `codex-rs/cli/src/main.rs:1259-1295` adds an
   `exec_server_config_path` resolver that walks `root_config_overrides`
   for the first non-`key=value` entry (treats it as a `--config PATH`
   override), falls back to `$CODEX_HOME/exec-server.toml`, then loads via
   `ExecServerConfig::load_from_path` and threads
   `into_run_options(&config_path)?` through the new
   `run_main_with_options` API. Three new tests at `:1840-1893` pin: no
   `--config` → `None`; explicit `--config /tmp/exec.toml` → `Some(path)`;
   `-c key=val` ignored. Default listen URL also moved to a
   `codex_exec_server::DEFAULT_LISTEN_URL` constant at `:453`.

3. **README contract.** `codex-rs/exec-server/README.md:26-58` documents
   `--config PATH`, the missing-file fallback, the `graceful_shutdown_timeout_ms`
   knob, the first-SIGINT-drains/second-SIGINT-forces semantics, and that
   websocket sessions can be resumed via `sessionId` after a connection
   close (i.e., the prior "close == terminate processes" behavior is
   replaced by detach + resume).

4. **Drain semantics** (per README):
   - First SIGINT/SIGTERM: stop accepting new websocket connections; reject
     new `process/start` and `http/request`; existing connections continue
     until processes/streams finish or `graceful_shutdown_timeout_ms`.
   - Second SIGINT/SIGTERM: force-stop all sessions.

5. **Cargo plumbing.** `codex-rs/exec-server/Cargo.toml:38/41` adds the
   `signal` tokio feature and a workspace `toml` dep.

## Why the change is right

The shape — "first signal → soft drain, timeout → force, second signal →
force immediately" — is the textbook supervisor-friendly shutdown contract
(matches systemd's `KillSignal`/`TimeoutStopSec`/`SendSIGKILL` shape and
nginx/postgres precedent). Splitting the new `--config PATH` parsing
*outside* the `-c key=value` namespace at `main.rs:1271-1284` is the right
ergonomic choice: the existing `-c k=v` overrides are general-purpose
(they apply across `codex` subcommands), while exec-server's TOML is a
dedicated server-config surface with its own schema, and conflating the
two would have made `--config` ambiguous.

`#[serde(deny_unknown_fields)]` on `ExecServerConfig` at `config.rs:24` is
the right default — typo'd keys fail loudly rather than silently using the
default timeout. The `DEFAULT_GRACEFUL_SHUTDOWN_TIMEOUT = Duration::from_secs(30)`
constant at `:6` is sane and matches the order-of-magnitude of a
process-cleanup window.

The test triplet at `main.rs:1853-1893` is the right contract pin: it
covers (a) default → no path, (b) explicit path → propagated, and
(c) `-c k=v` does not get misinterpreted as a path. Without the third arm
a future refactor that loosens the `if raw_override.contains('=')
{ continue; }` filter could silently break either feature.

## Nits (non-blocking)

1. **No test for `--config` collision detection.**
   `exec_server_config_path` at `main.rs:1281-1283` does
   `if config_path.replace(PathBuf::from(raw_override)).is_some() {
   anyhow::bail!("codex exec-server accepts at most one --config PATH"); }`,
   but no test asserts that two non-`=` overrides actually trigger the
   bail. Worth a 4th test arm so the user-facing error message is pinned.

2. **`graceful_shutdown_timeout_ms = 0` semantics undocumented.** The
   error variant comment ("must be greater than 0") implies validation,
   but the README only documents the default value. A user setting
   `graceful_shutdown_timeout_ms = 0` to mean "skip drain" would hit the
   validation error, but users wanting that semantic should be told to
   omit the key (default 30s) or use a tiny value like `1`. Worth a one-line
   README note.

3. **README behavioral change is buried.** The line "If the websocket
   connection closes, the server detaches from its session" at
   `README.md:50-52` is a *behavioral change* from the prior "terminates
   any remaining managed processes" — that's a significant API contract
   shift for any client that relied on close == cleanup. It deserves a
   `⚠ Behavior change` or a CHANGELOG entry pointing the migration path.

4. **No drain-timeout-elapses integration test in the diff.** The PR body
   names `exec-server-shutdown-test` as a bazel target but the diff
   itself only ships the CLI-arg-parse tests. A reviewer can't verify
   from the diff alone that the timeout-fires path actually force-stops
   in-flight processes; the named target may already cover this but a
   pointer in the PR description or a snippet of the assertion shape would
   help.

## Verdict rationale

Right-shaped supervisor-friendly shutdown overhaul with a clean config
surface, careful CLI-arg-parse separation of `--config PATH` from `-c
k=v`, and a behavioral contract documented in the README. The CLI-parse
tests are the strongest part of the diff. The behavioral-change-burial in
README and the missing collision/timeout-elapses tests keep this from
`merge-as-is`.

`merge-after-nits`
