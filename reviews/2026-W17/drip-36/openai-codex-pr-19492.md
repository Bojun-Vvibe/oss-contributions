# openai/codex #19492 — Streamline thread start handler

- **Repo**: openai/codex
- **PR**: [#19492](https://github.com/openai/codex/pull/19492)
- **Head SHA**: `2ecea71ab5e5b8b0061f61752770761278dbe72b`
- **Author**: pakrym-oai
- **State**: OPEN (+213 / -248, net -35)
- **Verdict**: `merge-as-is`

## Context

Same author / same handler-streamlining series as #19495, this time
targeting `thread/start`. Pre-refactor the start handler mixed (a)
request-shape validation, (b) config load + project-trust derivation,
(c) thread construction, (d) dynamic-tool validation, and (e)
JSON-RPC error emission in one ~250-line nested function.

## Design

Single-file change in
`codex-rs/app-server/src/codex_message_processor.rs`. Two structural
moves:

1. **Eager `send_error` for shape errors stays at the request
   boundary** (lines 14-19, 28-32 of the diff): the
   `permissionProfile cannot be combined with sandbox` and the
   `environment_selection_error_message(err)` paths still fire from
   the top-level handler. Good — these are pure validation, not
   embedded in the construction async block.

2. **The expensive setup is wrapped in `let result = async { ... }.await?;`**
   (line 55+). Inside that block, every fallible step now uses `?`:
   - `config_manager.load_with_overrides(...).await.map_err(|err| config_load_error(&err))?` (lines 57-60)
   - `set_project_trust_level(...)` falls back to in-memory overrides
     via the `cli_overrides_with_trust` branch (lines 86-114) — this
     is a logically-correct fallback: persisting trust is
     best-effort, and the in-memory `projects` table reconstruction
     at lines 96-113 mirrors what the on-disk version would have
     written. The `warn!` at line 92-94 surfaces the failure without
     blocking the thread start, which matches the pre-refactor
     behavior.

The `requested_permissions_trust_project` derivation at lines 67-78
correctly considers `WorkspaceWrite { .. }`, `DangerFullAccess`, and
`ExternalSandbox { .. }` as the trust-trigger sandbox policies. The
`ReadOnly` downgrade case mentioned in the comment (lines 62-66) is
the actual reason this exists — when enterprise config or Windows
forces ReadOnly, the user's *intent* (WorkspaceWrite) should still
mark the cwd trusted for the duration of the thread.

## Risks

- **Trust-persistence-failure path is now slightly more permissive in
  the failure mode**: previously the trust state was best-effort and
  failure left the project untrusted; the new code synthesizes a CLI
  override so the in-memory config behaves *as if* the project were
  trusted, even though disk state disagrees. This is the right
  behavior (otherwise the user sees their requested sandbox silently
  downgraded), but it deserves a comment on the divergence between
  in-memory and on-disk trust state. The `warn!` log line is
  enough for ops, but a code comment would help maintainers.
- **Net -35 lines** with no behavior change other than the above
  fallback path being explicit instead of implicit.

## Verdict

`merge-as-is` — same shape and quality as #19495 in the series.
