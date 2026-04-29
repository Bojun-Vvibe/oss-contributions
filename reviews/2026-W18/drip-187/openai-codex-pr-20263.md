---
pr: openai/codex#20263
sha: bb0795ef58659d4f44359e1286e90841af3ad32b
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# [codex] include originator on usage snapshot requests

URL: https://github.com/openai/codex/pull/20263
Files: `codex-rs/backend-client/src/client.rs`, `codex-rs/app-server/tests/suite/v2/rate_limits.rs`

## Context

The backend `Client` builds a `HeaderMap` in `client.rs:headers()` for
ChatGPT-account usage-snapshot requests (`/api/codex/usage`). All other
backend requests already carry an `originator` header (`codex-tui`,
`codex-vscode`, etc.) so the backend can attribute traffic and apply
per-surface rate-limit policies. The usage-snapshot path was the one
omission, which made it impossible to slice rate-limit telemetry by client
surface.

## What changed

`client.rs:204-208`: imports `originator` from `codex_login::default_client`
and inserts `h.insert("originator", originator().header_value)` as the first
header (before `USER_AGENT`). One line, no conditionals — if the originator
is unset, the existing `default_client::originator()` already returns a
sensible default (the same one all other clients in the repo use).

The integration test at `rate_limits.rs:152-167` switches from the bare
`mcp.initialize()` to `initialize_with_client_info(ClientInfo{name:
"codex-tui", ...})` and adds `.and(header("originator", "codex-tui"))` to
the wiremock matcher, asserting the header is now actually present on the
outbound request.

## What's good

- Symmetry with every other backend call site — the absence of this header
  was clearly an oversight, not a deliberate design choice.
- Test is a tight regression: it would have caught the missing header
  whether the bug was a forgotten `h.insert` or a typo in the header name.
- Zero risk surface: header addition is purely additive, the backend
  already accepts and routes on `originator` from sibling endpoints.

## Verdict

`merge-as-is` — trivial, well-scoped parity fix with a test that locks in
the contract. Nothing to discuss.
