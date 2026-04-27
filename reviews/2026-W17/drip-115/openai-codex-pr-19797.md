# openai/codex#19797 — Route MCP OAuth through runtime HTTP client

- PR: https://github.com/openai/codex/pull/19797
- Head SHA: `f8e2beea`
- Diff: +2494 / -544 across 25 files (largest hunks: `rmcp-client/src/mcp_oauth_http.rs` +690 new, `rmcp-client/src/oauth.rs` +416/-72, `rmcp-client/src/auth_status.rs` +128/-119, `codex-mcp/src/mcp/auth.rs` +527/-18)

## What it does
Plumbs MCP OAuth (discovery, dynamic client registration, PKCE token exchange, refresh) through the runtime-selected `HttpClient` instead of always going out from the local machine. Keeps browser interaction and the local callback listener on the user's machine (necessary — the user has to physically click "Approve") and keeps tokens in the existing local store. Net effect: when you select a remote runtime for MCP traffic, the OAuth side calls also egress from that runtime, so corporate egress controls / IP allowlists / proxy auth see a consistent origin.

Specific moves:
- `app-server` no longer depends on `codex-rmcp-client` directly (`Cargo.lock` and `app-server/Cargo.toml` deps removed). It now goes through `codex-mcp` helpers `perform_oauth_login_return_url_for_server` and friends — the OAuth orchestration that used to live in `codex_message_processor.rs` (lines 5941–5980 area in old code) is centralized in `codex-mcp`.
- New `mcp_oauth_http.rs` (+690) carries the OAuth-aware HTTP helper.
- `rmcp_client.rs` shrinks (+23/-49) — the OAuth-specific bits are now in the new module.
- New `tests/streamable_http_remote_oauth.rs` (+309) exercises the remote OAuth path including an unresolvable fake host (negative case) and an exec-server-backed stored-token refresh.

## Strengths
- The rationale ("OAuth side paths still used local HTTP, silently leaking origin") is real and not theoretical — exactly the kind of subtle inconsistency that breaks corporate deployments. The fix scope matches the diagnosis.
- Browser+callback are explicitly kept local. That's the only correct decision for OAuth UX (a remote runtime can't open the user's browser), and the PR is explicit about it.
- New regression tests cover both the happy path through a custom HttpClient and an unresolvable-host negative path. The exec-server-backed refresh test ensures the persisted-token branch also goes through the runtime client.
- Refactor reduces `app-server`'s direct dep on `rmcp-client`, which was the right architectural move — `app-server` should talk to `codex-mcp`, not reach into the rmcp internals.

## Concerns
- **Diff size (+2494/-544 across 25 files) is the dominant risk.** OAuth refactors in particular have a long tail of edge cases (token expiry races, refresh deduplication, store invalidation). The PR description mentions "serialized refresh attempts" was addressed in review feedback — that's exactly the kind of detail that benefits from a focused commit history. Suggest splitting into (a) introduce `mcp_oauth_http.rs` module and route discovery/registration, (b) move PKCE exchange + refresh, (c) move stored-token startup, (d) `app-server` cleanup. Reviewers can sign off each independently.
- **Test environment caveat in PR notes**: full `cargo test -p codex-core` and `-p codex-app-server` don't go green for the author (sandbox `Signal(6)` and a stack overflow in tracing tests). Need CI confirmation before merge.
- **`auth_status.rs` rewrite +128/-119**: that's nearly the whole file. Behavioral diff between old and new statuses (especially for the `mcp list` UX) deserves a one-paragraph note in the PR — what status string changes, if any, will users see?
- **`http_client_adapter.rs` +28/-4**: small but potentially user-visible (timeouts, retry policy). Worth confirming the adapter doesn't silently extend / shorten the OAuth call timeout vs the previous direct-reqwest path.
- The new `perform_oauth_login_return_url_for_server` helper takes `&server` directly (vs the old `(name, url, headers, env_headers, scopes)` decomposition). Slightly looser API contract; reasonable, but the helper should document which fields of `server` it actually reads (transport, scopes, http_headers, env_http_headers) so future server-config additions don't accidentally leak.

## Verdict
**needs-discussion** — the direction is correct and the test additions are good, but the diff is too large to merge without either (a) splitting into reviewable chunks or (b) at least one maintainer doing a careful end-to-end OAuth dry run against a real Streamable HTTP MCP server with refresh + revoke + expired-token edge cases. Don't merge on test coverage alone.
