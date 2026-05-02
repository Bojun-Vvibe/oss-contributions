---
repo: google-gemini/gemini-cli
pr: 26305
head_sha: 16364a4cf8f1a5a80e273c2c9afe5ffe2c7ef283
title: "feat(cli): add /mcp remove slash command for interactive server removal"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: google-gemini/gemini-cli#26305 — `add /mcp remove slash command`

**Head SHA:** `16364a4cf8f1a5a80e273c2c9afe5ffe2c7ef283`
**Stat:** +406 / −1 across docs + `packages/cli/src/ui/commands/mcpCommand{.ts,.test.ts}`.

## What it adds

Closes the asymmetry of the `/mcp` lifecycle: today users can
`/mcp add` (via settings file) and `/mcp enable | disable`, but to
*remove* an MCP server entry they have to drop out of the session and
hand-edit JSON. This PR adds:

```text
/mcp remove <name> [--scope user|workspace|all]
/mcp rm <name>          # alias
```

Behavior (per docs hunk in `docs/tools/mcp-server.md:1207–1227`):

- If the server is defined in only one scope (workspace OR user), it's
  removed without further prompting.
- If defined in **both** scopes, the command refuses and demands an
  explicit `--scope`. This is the correct safety choice — silently
  deleting only one would leave a "zombie" copy that re-activates on
  next restart, which is exactly the failure mode being warned against
  in the docs.
- Servers contributed by extensions are explicitly *not* removable —
  the doc directs users to disable the extension instead.
- After removal, the MCP client manager is restarted so the server's
  tools immediately disappear from the running session.

## Test coverage

`mcpCommand.test.ts:330+` adds a `describe('remove subcommand', ...)`
block that mocks `loadSettings` (top of file, lines 18–26) and
`forScope().settings.mcpServers`, and exercises:

- removal from a single-scope-defined server,
- the dual-scope refusal path,
- explicit `--scope user`, `--scope workspace`, `--scope all`,
- the "extension-contributed, refuse" path,
- the post-removal `restartMock` invocation.

The mock harness uses `installSettingsMock()` factory and
`scopeContents` map, which is the right shape to make later cases easy
to add.

## Assessment

- Solid feature design: the dual-scope refusal default is the safer
  behavior, and offering `--scope all` as the explicit override is
  better than asking twice.
- Restarting the MCP client manager post-removal is the right call —
  otherwise the tools keep working in the current session despite
  being "removed", which would be very confusing.
- Docs are updated in the same PR (good), with a clear example block.

## Nits

1. **`--scope all` semantics with extension-contributed servers.**
   What happens if a server is defined in user settings *and*
   contributed by an extension, and the user runs
   `/mcp remove foo --scope all`? Reading the docs hunk it's
   ambiguous — does it remove the user-settings copy and warn about
   the extension copy, or refuse entirely? Worth a test case and a
   doc clarification.

2. **No undo / dry-run.** Removal is destructive (writes settings
   file). A `--dry-run` flag that prints "would remove X from scope Y"
   without mutating would be friendly, especially for the
   `--scope all` case. Non-blocking.

3. **Restart cost.** Restarting the MCP client manager will
   transiently disconnect *every* MCP server, not just the one being
   removed. For users with many servers this may be a noticeable
   stall. Worth measuring; if it's >1s the docs should mention it,
   and a future enhancement could selectively disconnect only the
   removed server. Out of scope for this PR.

4. **Alias surface.** Aliasing `/mcp rm` is good. Consider also
   `/mcp delete` and `/mcp del` for muscle memory from other CLIs.
   Bikeshed-tier; ignore if maintainers prefer the minimal surface.

## Verdict

**`merge-after-nits`** — well-designed feature that closes a real
gap, with proper test coverage and docs in the same PR. Nit #1 is
the one I'd actually want resolved (extension+user-scope interaction
needs a defined behavior); the rest are polish/follow-ups.
