# openai/codex PR #20718 — Add app-server daemon lifecycle management

- PR: https://github.com/openai/codex/pull/20718
- Head SHA: `e9d25cbffb14313304c543c852d68e6896d1d8af`
- Author: @euroelessar (Ruslan Nigmatullin)

## Summary

Adds a new `codex-app-server-daemon` crate and wires it into `codex app-server` lifecycle subcommands (`start`, `restart`, `stop`, `version`, `bootstrap`) so desktop / mobile clients reaching a remote machine over SSH can bootstrap and manage `codex app-server` programmatically. All lifecycle commands emit JSON to stdout. Backend selection prefers user-scoped `systemd` and falls back to a pidfile-backed detached process. `bootstrap` supports the standalone managed install with home-scoped systemd units and an hourly auto-update timer. Lifecycle ops are serialized per-`CODEX_HOME` via a `daemon.lock` and distinguish graceful reload (drain forever) from forced restart (drain ~60s).

## Specific references from the diff

- New crate `codex-rs/app-server-daemon/` (Cargo manifest, `BUILD.bazel`, `README.md`, `src/backend/mod.rs` with backend trait, plus systemd + pidfile backends implied by the README and Cargo deps on `tokio` features `["fs","macros","process","rt-multi-thread","time"]` and `tokio-tungstenite`).
- `codex-rs/Cargo.toml` workspace: adds `app-server-daemon` to `members` and `codex-app-server-daemon = { path = "app-server-daemon" }` to `[workspace.dependencies]`.
- `codex-rs/Cargo.lock` records the new crate and adds it as a dependency of the top-level binary crate (the diff shows it appearing in the dependency list right after `codex-app-server`).
- `codex-rs/app-server-daemon/README.md` documents `start`/`restart`/`stop`/`version`/`bootstrap`, the `CODEX_HOME/app-server-daemon/` state layout (`settings.json`, `app-server.pid`, `daemon.lock`), and the `codex-app-server-<hash>.service` naming scheme. Author claims live validation on `devbox-ruslan` (systemd) and `cb4` (pidfile).

## Verdict: `needs-discussion`

The change itself is well-scoped and the lifecycle semantics are thoughtfully laid out, but a new daemon crate that talks to user-systemd, manages pidfiles, owns an hourly auto-update timer, and is intended to be invoked over SSH on someone else's machine is a meaningful trust-surface expansion. That belongs in a design discussion, not a single PR review.

## Nits / concerns

1. **Auto-update timer is silently installed by `bootstrap`.** The README describes an hourly timer that "refreshes the standalone install and then issues `systemctl --user reload`" — i.e. the daemon will quietly upgrade itself in the background on every host that ever runs `bootstrap --remote-control`. That should be opt-in (`--enable-auto-update`) or at least loud in the bootstrap JSON response, with a documented `disable` path. The current shape makes "I ran one curl | sh + bootstrap on a dev VM" turn into a long-lived self-updating service.
2. **Pidfile backend lifecycle on signal-9 / OOM.** `daemon.lock` + `app-server.pid` need a clear story for "process was hard-killed and never cleaned up". The README mentions "stale state cleanup" but the diff section shown doesn't include that logic — please point reviewers at the exact module (e.g. `src/backend/pidfile.rs`) and the test that exercises stale-pidfile reclamation. Without it, `start` after an OOM kill could refuse to come up.
3. **Concurrency story across multiple `CODEX_HOME` roots.** README says lifecycle ops are serialized "per `CODEX_HOME`" via `daemon.lock`. Worth confirming that two different `CODEX_HOME` values on the same user account get distinct unit names *and* distinct lock files — the unit-name hash story is in the README but I'd want a unit test asserting `lock_path(home_a) != lock_path(home_b)` to lock that contract in.
