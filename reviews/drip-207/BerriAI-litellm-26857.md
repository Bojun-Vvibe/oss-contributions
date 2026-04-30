# BerriAI/litellm #26857 — feat(mcp): support stateless and stateful clients via session-id routing

- Head SHA: `6b470bc3eba5a7eaa8526f53c6b122589b63bd4a`
- Files: `litellm/proxy/_experimental/mcp_server/server.py`, `tests/mcp_tests/test_mcp_server.py`, `tests/test_litellm/proxy/_experimental/mcp_server/auth/test_user_api_key_auth_mcp.py`, `tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server.py`, `tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_stale_session.py`
- Size: +243 / -40

## What it does

Fixes LIT-2196 / LIT-2656. Today litellm's MCP proxy runs a single
`StreamableHTTPSessionManager` with `stateless=True` (regression-fixed
in PR #19809 to unbreak curl / MCP Inspector / any HTTP client that
doesn't manage `mcp-session-id`). But `stateless=True` also breaks
progress notifications and any flow that needs the server to assign
and remember a session ID — meaning Claude Code, Cursor, and VSCode
MCP clients can't get long-running tool progress updates.

This PR runs **both** managers side-by-side and routes per-request:

- `server.py:238-248` instantiates `session_manager_stateless`
  (stateless=True) and `session_manager_stateful` (stateless=False).
  The old `session_manager` name is preserved as an alias to the
  stateless one (`:252`) so any external monkeypatch that hardcoded
  the name keeps working — though the in-tree test files all migrate
  to the new explicit names.
- Lifecycle (`initialize_session_managers` /
  `shutdown_session_managers` at `:268-318`) gains a third context
  manager `_session_manager_stateful_cm` and `__aenter__`s/`__aexit__`s
  it alongside the existing two. The exception handler stays as a
  single try/except, so a partial-failure during init can still leave
  one of the three managers in a half-started state — pre-PR behavior,
  not a regression.
- Two new helper closures inside `handle_streamable_http_mcp`:
  `_get_session_id_from_scope` (`server.py:2524-2538`) reads
  `mcp-session-id` from the ASGI headers handling both `bytes` and
  `str` header values; `_is_initialize_request` (`server.py:2540-2554`)
  parses the request body as JSON and returns True iff
  `data.get("method") == "initialize"`.
- Routing decision at `server.py:2805-2825`: peek the first
  `receive()` message **only when** `method == "POST"` AND no
  `mcp-session-id` header. If `session_id` is present OR the body is
  an `initialize` request → route to stateful; else → stateless. The
  consumed first chunk is replayed via a `wrapped_receive` closure
  (`:2828-2840`) that returns the cached message once and then
  delegates to the original `receive` thereafter.
- Stale-session handler call at `:2849` and the final dispatch at
  `:2854` both swap `session_manager` for the computed
  `target_manager` so the stale-session DELETE-idempotency path also
  routes to the right manager.

Tests cover all three behaviors:
`test_streamable_http_session_manager_is_stateless` at
`test_mcp_server.py:937-957` now asserts both
`session_manager_stateless.stateless is True` AND
`session_manager_stateful.stateless is False`. New
`test_mcp_routing_initialize_to_stateful_no_session_to_stateless` at
`:959-1023` POSTs both an `initialize` body and a `tools/list` body
without a session-id and asserts the routing dispatches to the
expected manager only. The stale-session test at
`test_mcp_stale_session.py:341-395` is updated to monkeypatch
`session_manager_stateful` (since requests with `mcp-session-id`
route there).

## What works

The "peek then replay" pattern at `server.py:2828-2840` is the right
shape for ASGI request inspection: you can't `await receive()` twice,
so wrapping it in a closure that returns the cached first chunk on
the first call and then transparently delegates is the standard
solution. The `nonlocal first_msg` capture-then-clear ensures replay
happens exactly once per request.

The routing condition is well-chosen and minimal: `has session_id OR
is initialize → stateful`. The `initialize`-without-session-id case
correctly routes to stateful so that the server *creates* a session
ID for the response, which is exactly what stateful clients (Claude
Code, etc.) need on first contact. Stateless clients (curl,
Inspector) never send `initialize` (they just POST `tools/list` etc.
directly) so they continue to land on the stateless manager and don't
need a session.

Keeping `session_manager` as an alias (`server.py:252`) is the right
backward-compat call: any plugin or third-party code that
monkeypatches the old name still gets the stateless manager (which
is the pre-PR behavior, since pre-PR there was only the stateless
one). Internal call sites all migrate to explicit names.

The `# noqa: PLR0915` at `:2702` honestly acknowledges the
too-many-statements lint hit rather than swallowing it — better than
silently splitting the function for the linter and making the
control flow harder to read.

## Concerns

1. **Body-peek assumes single-chunk receive.** The PR comment at
   `server.py:2545-2547` openly notes "Assumes the body fits in a
   single ASGI receive() chunk." For an MCP `initialize` request
   that's safe (the body is tiny), but the routing decision is made
   against an *uninspected* body — a maliciously chunked body where
   the `method` field straddles the chunk boundary would parse as
   non-JSON, return False from `_is_initialize_request`, and route
   to stateless. That's the safe failure direction (worst case is
   "client hits stateless and gets the same behavior they got
   pre-PR"), but worth a `more_body=True` check that escalates to
   stateful as the conservative default if we can't tell.
2. **First-message replay drops `more_body` semantics.** The
   `wrapped_receive` closure at `:2828-2840` returns the cached
   first message verbatim, but if that first chunk had
   `more_body=False` (a complete request) the downstream consumer
   sees the same `more_body=False` and won't call `receive()` again
   — which is correct. If `more_body=True`, the downstream consumer
   calls `receive()` and falls through to `original_receive` —
   which is also correct. So the contract holds. But there's no test
   for the `more_body=True` chunked case — the new routing test
   uses a single-chunk body. Worth one more test where the first
   chunk has `more_body=True`.
3. **Concurrent-init regression test only checks call counts, not
   ordering.** `test_concurrent_initialize_session_managers` at
   `test_mcp_server.py:853-942` correctly asserts each manager's
   `.run()` is called exactly once and `__aenter__` is called three
   times total. But if init partially fails (e.g., stateful manager's
   `__aenter__` raises), the existing exception handler at
   `:288-307` will leak the *already-entered* stateless manager
   without ever calling `__aexit__`. Pre-PR same issue for two
   managers; this PR triples the surface. A try/except-with-cleanup
   refactor would be the right follow-up.
4. **`session_manager` alias is a footgun.** Keeping
   `session_manager = session_manager_stateless` for backward-compat
   is correct, but a future contributor reading the file will assume
   "the session manager" is *the* one — and any new code that
   references `session_manager` will silently bypass the routing.
   A `# DEPRECATED: alias for backward-compat, use _stateless or
   _stateful explicitly` comment at `:252` would help.
5. **`set_auth_context` for the dual managers.** The diff doesn't
   show whether auth context is correctly attached to both
   managers' running tasks. If it's stored in a contextvar that's
   bound to the stateless manager's task pool, requests routed to
   stateful might see stale or missing auth. Worth verifying that
   the auth-context plumbing at `extract_mcp_auth_context` is
   manager-agnostic — the existing test
   `test_user_api_key_auth_mcp.py:1520-1530` patches both names so
   presumably this is fine, but the PR description doesn't call it
   out.

Verdict: merge-after-nits

## What I learned

ASGI's "you can only `await receive()` once" rule pushes you toward
a peek-and-replay closure pattern whenever you need to make a
routing decision based on body content. The closure version
(`wrapped_receive` here) is strictly better than the alternative of
*reading the entire body up-front and storing it* because it
preserves the streaming contract — the downstream consumer still
calls `receive()` lazily and can stop early if it wants. The cost
is one captured variable and four lines of plumbing per peek site,
which is cheap. The discipline that matters is making the peek
*idempotent* (only happens on POST + no-session-id) so that the
common path doesn't pay the consume-and-replay cost; this PR gets
that right at `server.py:2810`.
