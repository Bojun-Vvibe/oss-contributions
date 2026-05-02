# openai/codex PR #20665 — Make environment providers own default selection

- Head SHA: `4682df92650f905a6d2b085053d2b97f46fb0cfd`
- URL: https://github.com/openai/codex/pull/20665
- Size: +175 / -25, 3 files (codex-rs/exec-server)
- Verdict: **merge-as-is**

## What changes

Refactors `EnvironmentManager` so that the *default environment
selection policy* is owned by the `EnvironmentProvider`, not by
`EnvironmentManager` itself.

- `environment_provider.rs:31-37` introduces the
  `DefaultEnvironmentSelection` enum: `Derived` (current heuristic),
  `Environment(String)` (explicit), `Disabled` (force `None`).
- `EnvironmentProvider` gains a default-implemented
  `default_environment_selection(&self) -> DefaultEnvironmentSelection`
  that returns `Derived`, so existing impls don't break.
- `DefaultEnvironmentProvider::default_environment_selection`
  (environment_provider.rs:84-91) reads the `CODEX_EXEC_SERVER_URL=none`
  legacy crutch and returns `Disabled` — moving that branch out of
  `EnvironmentManager` and behind the trait it belongs to.
- `EnvironmentManager::from_environments` (environment.rs:155-184) now
  takes a `DefaultEnvironmentSelection` and either derives, validates
  the explicit id, or disables. Returns `Result<Self, ExecServerError>`
  so an unknown explicit id surfaces as a `Protocol` error instead of
  silently falling back.
- `Environment::remote_with_transport` (environment.rs:184-197) is
  added so non-WebSocket transports (`StdioShellCommand`) can construct
  remote environments without going through `WebSocketUrl`. The new
  `remote_transport: Option<ExecServerTransport>` field replaces the
  url-based `is_remote()` check (environment.rs:347).

## What's good

- The previous `EnvironmentManager::from_default_provider` had a
  comment "TODO: Remove this legacy `CODEX_EXEC_SERVER_URL=none`
  crutch once environment attachment defaulting moves out of
  EnvironmentManager" (visible in the - hunk at environment.rs:108).
  This PR is exactly that move. The TODO is satisfied and the
  responsibility ends up where it belongs (in the provider).
- Three new tests cover the three new variants:
  `environment_manager_uses_explicit_provider_default`
  (environment.rs:237-255),
  `environment_manager_disables_provider_default`
  (environment.rs:259-274), and
  `environment_manager_rejects_unknown_provider_default`
  (environment.rs:277-292) — that last one checks the actual error
  string ("default environment `missing` is not configured"),
  not just the variant.
- `is_remote()` switching from `exec_server_url.is_some()` to
  `remote_transport.is_some()` (environment.rs:347-348) is the right
  fix: with the new stdio transport, a remote environment legitimately
  has no URL, and the old check would have wrongly classified it as
  local.
- The `disabled_for_tests` panic (environment.rs:75-83) is appropriate
  for a test-only helper; don't see a need for a `Result` there.

## Nits (non-blocking)

- `Environment::remote_with_transport` is `fn` (private) but
  `Environment::remote` (the public-ish flavor) just delegates to it.
  If a downstream provider wants to construct a stdio remote, it'll
  have to add another constructor. That can wait for the first
  consumer.
- `default_environment_selection` returns by value (`Clone` on
  `String` inside `Environment(String)` variant). At the call sites
  this is fine, but if the explicit id ever becomes hot-path
  (per-turn) consider taking `&str` or returning a `Cow`.

## Risk

Low. Behavior preservation is good: every existing path still ends up
in `Derived` (the trait default + `DefaultEnvironmentProvider`'s
override that goes `Derived` unless `CODEX_EXEC_SERVER_URL=none`).
The new error path is reachable only when a custom provider returns
`Environment("missing")`, which is exactly what should fail loudly.

The transport-vs-url separation in `Environment` (storing both
`exec_server_url: Option<String>` *and* `remote_transport:
Option<ExecServerTransport>`) is a temporary redundancy — the URL is
derivable from the transport. Worth a follow-up to drop the cached
`exec_server_url` field once all readers go through the transport.
