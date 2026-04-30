# Review: aaif-goose/goose#8925 — Added recipe discovery / execution to ACP server

- PR: https://github.com/aaif-goose/goose/pull/8925 (originally listed under block/goose#8925 — repo
  has been transferred to aaif-goose org)
- Author: natelastname
- headRefOid: `88955a934d59b2b3611b0d3200d633f9155265d9`
- Files: `crates/goose/src/acp/server.rs` (+95/-16)
- Verdict: **merge-after-nits**

## Analysis

Adds two related ACP capabilities to the Goose ACP server. First, on session creation and load
(call sites at `acp/server.rs:1822` and `acp/server.rs:2236`), the server now emits a
`SessionUpdate::AvailableCommandsUpdate` notification listing every entry in the user's configured
`slash_commands` table. The list is built by `build_available_commands_from_slash_commands`
(server.rs:1576-1599) which walks `crate::slash_commands::list_commands()`, reads the recipe file
from disk, parses it via `Recipe::from_content`, and constructs an `AvailableCommand` with the
recipe's `description` and (optionally) an unstructured input hint derived from the recipe's
parameter list via `command_input_hint` (server.rs:1565-1573). The hint heuristic prefers a
parameter named `args`, then the first parameter without a default, then the first parameter — a
reasonable hierarchy for picking the "free-text input" slot.

Second, at the prompt-handling site (server.rs:2280-2298), if the first whitespace-separated token
of the user message starts with `/` and resolves to a known slash command, the server emits an
`AgentMessageChunk` containing `Running recipe: /<command>` so the user has visible feedback that
the recipe was matched (recipe expansion itself is otherwise agent-only).

**Nit 1 (worth addressing):** No tests. The PR author offers to add some if the approach is
accepted, which is the right offer — at minimum I'd want unit coverage for `command_input_hint`
(args-vs-default-vs-first precedence) and an integration smoke test that
`build_available_commands_from_slash_commands` skips entries whose `recipe_path` doesn't exist
without panicking. Both are pure-Rust and shouldn't require an ACP harness.

**Nit 2 (potential bug):** `command_input_hint` (server.rs:1567-1573) clones
`p.description` unconditionally. If the recipe parameter has an empty description, the
`UnstructuredCommandInput` will have an empty hint string, which the client may render as an empty
hint bubble. Consider `Some(p.description.clone()).filter(|s| !s.is_empty())`.

**Nit 3 (matching too eager):** The `/`-token matcher (server.rs:2284-2297) checks
`message_text.trim().split_whitespace().next()`, which means a message like `/help me with this`
where `/help` happens to be a registered slash command would emit "Running recipe: /help" even if
the user intended natural-language usage. That's fine for genuine slash commands but worth being
intentional about — `slash_commands` table membership is the gate, so as long as users only
register actual recipe shortcuts there, the false-positive surface is small.

**Nit 4 (perf):** Every session creation re-reads and re-parses every recipe file from disk
(server.rs:1581-1582). For a user with many slash commands this is a non-trivial sync I/O hit on
the session-establishment path. Worth caching the parsed list (with mtime invalidation) if this
shows up as a startup latency issue, but premature for a first cut.

The intent is right and the diff is well-localized. Merge once at least one happy-path unit test
lands and the empty-description case is guarded.
