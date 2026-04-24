# codex PR #19193 — Support Codex Apps auth elicitations

Link: https://github.com/openai/codex/pull/19193

## What it changes

When a Codex Apps tool call fails because the underlying connector
needs auth (the response carries connector-auth metadata), this PR:

- requests a URL-mode MCP elicitation back through the
  `mcp_connection_manager`,
- routes Codex Apps auth-URL elicitations into the TUI's
  `app_link_view` flow (the existing app-link UX previously used for
  plugin install URLs),
- advertises URL elicitation support from the MCP client so servers
  know they may send the metadata,
- adds tests around `mcp_tool_call`, the connection manager, and the
  TUI app-link rendering for the new auth-URL case.

945 added / 54 deleted, with the bulk in `core/src/mcp_tool_call.rs`
(+299) and `tui/src/bottom_pane/app_link_view.rs` (+430). Unrelated
test failures in `core` / `tui` are flagged as environment-driven.

## Critique

The architectural shape is right: piggyback on the existing
`app_link_view` for the auth flow rather than inventing a parallel UI
surface, and use MCP elicitations as the wire mechanism rather than
inventing a Codex-specific signal. Two correctness concerns:

1. **Elicitation-support advertisement is a forward contract.** Once
   the MCP client advertises URL-elicitation capability, servers
   may send these elicitations on *any* tool call, not only Apps
   tool calls. The PR scopes routing in `chatwidget.rs` /
   `thread_routing.rs` to "Apps auth metadata," but a misbehaving
   or experimental MCP server could now send a URL elicitation in
   response to a normal tool call, expecting the same UX. The PR
   should specify what happens in that case — drop with a log,
   surface as a generic app-link, or error — and add a test for
   the "unexpected URL elicitation from non-Apps server" path.
2. **app_link_view +430 is a lot for a single view.** The view now
   has to differentiate plugin-install URLs from auth-URL flows;
   the snapshot test added
   (`app_link_view_generic_auth_url_elicitation.snap`) is one
   shape but the matrix (auth + install, auth-only, install-only,
   neither) deserves four snapshots, not one.
3. **Snapshot-only coverage for the TUI logic.** Snapshot tests
   confirm the rendering does not regress, but do not assert
   that selecting "Open" actually invokes the auth URL with the
   expected parameters. A behavioral test on `app_link_view` —
   simulate the "Open" action and assert the dispatched event
   carries the URL — would catch logic regressions snapshots miss.

## Suggestions

- Add an explicit fallback path for "URL elicitation from a
  non-Apps tool call" (recommend: log + show generic app-link)
  with a focused test.
- Extend the snapshot matrix to the four (auth × install)
  combinations.
- Add a behavioral "Open invokes auth URL" assertion alongside the
  snapshot.

Verdict: thoughtful integration. Add the unexpected-elicitation
fallback test before merging.
