# codex PR #19240 — fix: allow AgentIdentity through Apps MCP gates

Link: https://github.com/openai/codex/pull/19240

## What it changes

Three small enablement gates in the Apps MCP refresh path were still
hard-coded to "must be ChatGPT token auth" rather than the new
`AuthProvider`-driven model. After PR #18811 routed backend auth
through `AuthProvider` and PR #18904 added AgentIdentity JWT support,
those three sites still returned an empty connector list / skipped
plugin auth follow-up for AgentIdentity-backed sessions. This PR
swaps each gate to call `CodexAuth::uses_codex_backend` instead.

Six lines changed across:

- `core/src/connectors.rs::list_cached_accessible_connectors_from_mcp_tools`
- `core/src/connectors.rs::list_accessible_connectors_from_mcp_tools_with_environment_manager`
- `app-server/src/codex_message_processor/plugins.rs`

## Critique

This is the correct shape of fix: a parity bug where a new auth
identity was introduced but a few gate predicates still encoded the
old ChatGPT-only assumption. The bug class is recognizable —
"refactored auth at the producer side, forgot a few consumer-side
gates" — and the right cure is to centralize the predicate, which
this PR does by routing all three sites to a single helper.

Concerns:

1. **Are there other gates?** The PR enumerates three call sites by
   inspection. A grep for `is_chatgpt_token` (or whatever the prior
   predicate was named) across `codex-rs` should confirm no fourth
   site escaped. If the grep returns more, the PR is under-scoped.
   Worth either listing the grep result in the PR description or
   adding an `#[allow(dead_code)]` removal of the old predicate to
   force a compile-time error if any caller remains.
2. **Test coverage.** The PR mentions formatting + scoped clippy +
   scoped cargo check, but no functional test that an AgentIdentity-
   authed session now sees a non-empty connector list from
   `list_cached_accessible_connectors_from_mcp_tools`. Without that
   test the next refactor of `uses_codex_backend` could silently
   regress AgentIdentity again.
3. **Semantics of `uses_codex_backend`.** The fix assumes
   `uses_codex_backend` is the *correct* predicate — i.e., that the
   intent of the original gate was "this auth talks to the Codex
   backend" rather than "this auth is specifically a ChatGPT user
   token." If a future identity (e.g., a service account) also
   `uses_codex_backend` but should *not* see Apps connectors, the
   predicate is too coarse. A comment at each call site explaining
   *why* the gate exists would forestall that.

## Suggestions

- Add a focused integration test under
  `core/tests/connectors_agent_identity.rs` that constructs an
  AgentIdentity-authed `AuthProvider` and asserts the connector list
  is populated.
- Either delete the prior predicate to force a compile error on any
  remaining caller, or add a grep result to the PR body documenting
  the call-site enumeration.

Verdict: small, correct ACL-parity fix. Land with one regression test.
