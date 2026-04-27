# openai/codex #19797 — Route MCP OAuth through runtime HTTP client

- **Repo**: openai/codex
- **PR**: #19797
- **Author**: aibrahim-oai
- **Head SHA**: a4de53c661aa52371b0d9ad28c683c49528b61a9
- **Size**: +1500 / −352 across ~25 files (substantial OAuth subsystem refactor)

## What it changes

Threads the codex runtime HTTP client through the entire MCP
OAuth login + discovery surface, replacing ad-hoc `reqwest`
clients (one per call site) with a single
`Arc<dyn HttpClient>` constructed once per server config from
the active environment manager.

Three structural moves:

1. **New shared constructor** in `codex-mcp/src/mcp/auth.rs:355-358`:

   ```rust
   pub fn http_client_for_server(
       config: &McpServerConfig,
       runtime_environment: McpRuntimeEnvironment,
   ) -> Result<Arc<dyn HttpClient>>
   ```

   Re-exported from `codex-mcp::lib` (line 213/474) and
   `codex_mcp::http_client_for_server` is now the single
   source of OAuth-bound HTTP clients.

2. **API surface widening** in `codex-rmcp-client`. Three
   functions get `_with_client` siblings that take an
   explicit `Arc<dyn HttpClient>`:
   - `discover_supported_scopes(transport)` →
     `discover_supported_scopes(transport, http_client)`
     at `codex-mcp/src/mcp/auth.rs:272`
   - `perform_oauth_login_silent` →
     `perform_oauth_login_silent_with_client` at
     `rmcp-client/src/perform_oauth_login.rs:2033`
   - `perform_oauth_login_return_url` →
     `perform_oauth_login_return_url_with_client` at
     `rmcp-client/src/perform_oauth_login.rs:2093`

   The plain (non-`_with_client`) variants are retained for
   any caller who hasn't migrated yet — but the four call
   sites this PR touches (`codex_message_processor.rs:5961`,
   `plugin_mcp_oauth.rs:23`, `cli/src/mcp_cmd.rs:323/417`,
   plus `core/src/mcp_skill_dependencies.rs:128`) all switch.

3. **Construction discipline** is identical at each call
   site and worth pinning verbatim:

   ```rust
   let environment_manager = self.thread_manager.environment_manager();
   let runtime_environment = match environment_manager.default_environment() {
       Some(env) => McpRuntimeEnvironment::new(env, config.cwd.to_path_buf()),
       None     => McpRuntimeEnvironment::new(
                       environment_manager.local_environment(),
                       config.cwd.to_path_buf()),
   };
   let http_client = http_client_for_server(server, runtime_environment)?;
   ```

   This default→local fallback is repeated five times
   (call sites listed above). Worth extracting into
   `EnvironmentManager::default_or_local(cwd)` in a follow-up.

## Strengths

- **Right architecture.** OAuth discovery and login are now
  network-policy-bound the same way every other MCP HTTP
  call is — proxy settings, TLS roots, retry policy,
  trace-context injection all flow through the one
  `HttpClient`. Pre-fix, `discover_supported_scopes` was
  cutting a fresh `reqwest::Client` per call with no
  proxy/TLS/headers awareness, which broke OAuth in any
  corp-proxy or custom-CA environment without a clear
  failure signal.
- **Disciplined feature additions vs renames.** The
  `_with_client` suffix is the standard idiom in this
  codebase (consistent with prior PRs) and the legacy
  signatures stay around as thin wrappers — no
  big-bang break for downstream consumers like
  `cli/src/mcp_cmd.rs` callers maintained out-of-tree.
- **Strong test coverage.** New tests at
  `codex-mcp/src/mcp/auth.rs:399-460` (`#[test]`,
  `#[test]`, `#[tokio::test]`) pin three cases:
  the `local_environment()` fallback succeeds for a
  simple stdio config, the `default_environment()`
  path succeeds for a streamable-http config, and the
  third asserts a specific error class for a remote
  runtime environment with bad inputs. Plus a +309-line
  integration test at
  `rmcp-client/tests/streamable_http_remote_oauth.rs`
  that exercises the end-to-end discovery+login round-trip
  against the in-tree `test_streamable_http_server` binary.
- The `Cargo.lock` change at `codex-rs/Cargo.lock:3168`
  drops `codex-client` from the rmcp-client dependency
  list, confirming this also removes a redundant HTTP
  stack and shrinks the dep graph by one crate.

## Concerns / nits

- **Five-fold duplication of the env-resolution snippet.**
  The `default_environment().unwrap_or(local)` boilerplate
  appears at `codex_message_processor.rs:5961`,
  `plugin_mcp_oauth.rs:23/107`, `mcp_cmd.rs` (twice via
  the `:323`/`:417` arms), and `mcp_skill_dependencies.rs`.
  Should be one
  `pub fn default_or_local_for(&self, cwd: &Path) -> McpRuntimeEnvironment`
  method on `EnvironmentManager`. Otherwise a future env
  semantic (e.g. "prefer plugin-scoped env if available")
  has to be added in five places and any drift is silent.
- **Error message inconsistency.** The CMP arm at
  `codex_message_processor.rs:5970-5979` produces a JSON-RPC
  `INVALID_REQUEST_ERROR_CODE` with a stringified `err`,
  while the plugin path at `plugin_mcp_oauth.rs:31` just
  `warn!`s and `continue`s. The plugin path silently
  proceeding past a misconfigured environment has no
  user-visible signal — should at minimum emit a
  `ServerNotification` so the TUI can surface "plugin X's
  MCP OAuth could not be configured" rather than appearing
  to install successfully.
- **Test for the `discover_supported_scopes` 404/non-200
  path with the new `http_client` parameter is not visible
  in this diff.** Pre-fix, that function returned `None`
  on any error; post-fix, errors from the runtime client
  (proxy denials, TLS failures) might propagate
  differently. A pin test that the function still returns
  `None` on a 404 from the well-known endpoint and on a
  proxy-denied 407 would lock the caller-friendly contract.
- The `compute_auth_status` arm at `auth.rs:194-262` adds
  a new `runtime_environment` parameter through the
  `compute_auth_statuses` iterator API at `:144-163`,
  which is a public function (`pub async fn`). External
  callers (anyone using `codex-mcp` as a library for
  status reporting) will get a compile break. Worth a
  CHANGELOG line.

## Verdict

**merge-after-nits.** Architecturally correct and
well-tested — this is the right way to plumb OAuth
through the runtime HTTP stack and closes a real
corp-proxy/custom-CA gap. Two pre-merge items: factor the
five-fold env-resolution snippet into one
`EnvironmentManager` method (or accept a follow-up PR
note), and surface the plugin-OAuth-env-failure case as
a `ServerNotification` rather than a silent log warn.

## What I learned

When a feature subsystem (here: MCP OAuth) cuts its own
HTTP client per call site, every cross-cutting concern
the rest of the codebase has solved (proxy resolution,
TLS roots, trace-context injection, retry policy)
silently fails to apply, and the failure mode is "works
on my machine, breaks in corp environments with no
useful error". The fix is mechanical but high-value:
identify the shared abstraction (`HttpClient`),
construct it once at the call boundary using the same
runtime context every other subsystem uses
(`EnvironmentManager`), and thread it through. The
five-fold boilerplate at construction sites is a smell
worth refactoring before the boilerplate calcifies.
