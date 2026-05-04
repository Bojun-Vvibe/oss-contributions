# openai/codex#21059 — Rename agent identity login surface to access token

- **URL**: https://github.com/openai/codex/pull/21059
- **Head SHA**: `f7f73ce43d81`
- **Diffstat**: +144 / -69
- **Verdict**: `merge-after-nits`

## Summary

Renames the external login surface from "Agent Identity" to "access token" terminology. Adds `CODEX_ACCESS_TOKEN` env var (with `CODEX_AGENT_IDENTITY` retained as fallback) and `codex login --with-access-token` CLI flag (with `--with-agent-identity` hidden as a back-compat alias). Internal `AuthMode::AgentIdentity` is intentionally left in place — only the user-facing surface is renamed.

## Findings

- `codex-rs/cli/src/lib.rs:13,16` — re-exports renamed: `read_agent_identity_from_stdin → read_access_token_from_stdin`, `run_login_with_agent_identity → run_login_with_access_token`. This is a public-API break for any external Rust consumer of `codex-cli` as a library. Worth a one-line CHANGELOG entry.
- `codex-rs/cli/src/login.rs:38-39` — `AGENT_IDENTITY_LOGIN_DISABLED_MESSAGE` const renamed to `ACCESS_TOKEN_LOGIN_DISABLED_MESSAGE` and message body updated. Clean.
- `codex-rs/cli/src/login.rs:193-219` — `run_login_with_access_token` body unchanged in shape; only the parameter name (`agent_identity → access_token`), tracing message, and error string updated. Underlying `login_with_access_token()` call delegates to the renamed `codex-login` crate function.
- `codex-rs/cli/src/login.rs:233-239` — `read_access_token_from_stdin()` updates the user-facing prompt to reference both `--with-access-token` and `CODEX_ACCESS_TOKEN`. Good UX wording.
- `codex-rs/cli/src/login.rs:391` (login status path) — when status reports `AuthMode::AgentIdentity`, the printed string is renamed to "Logged in using access token" (per the truncated diff). Internal enum variant kept, so on-disk auth files written under the old name still display correctly after upgrade. Confirmed by the test additions in `codex-rs/login/src/auth/auth_tests.rs` (+61 / -28).
- `codex-rs/login/src/auth/manager.rs:+33 / -11` — adds env precedence logic. PR body claims `CODEX_ACCESS_TOKEN` wins over `CODEX_AGENT_IDENTITY`; would be useful to verify this is also covered by an explicit "both set, new wins" test case in `auth_tests.rs` (the +61 lines of test additions suggest yes, but worth a reviewer spot-check).
- `codex-rs/cli/tests/login.rs:+3 / -3` — only the renamed test names / strings updated. Light coverage for the new CLI flag itself; ideally one end-to-end `--with-access-token` invocation test should exist (may already be there in the +61 auth_tests block).

## Nits

- Hide `--with-agent-identity` from `--help` but keep accepting it for at least one minor version; a deprecation note in `--help` text for `--with-access-token` ("alias: --with-agent-identity, deprecated") would help discovery for users searching for the old flag.
- The bare term "access token" is somewhat ambiguous (OAuth access token? OpenAI API key? agent-scoped token?). Consider `--with-codex-access-token` for the long form, or at least a one-liner in the `--help` clarifying scope.

## Recommendation

Clean rename, internal/external boundary maintained, fallbacks preserved. Address the help-text nits and add a CHANGELOG entry for the public Rust re-export rename, then merge.
