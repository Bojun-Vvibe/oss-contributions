# sst/opencode #25649 — fix: increase LSP initialize timeout for JDTLS and KotlinLS

- **PR**: https://github.com/sst/opencode/pull/25649
- **Head SHA**: `9e147ef185e25d5f4af7ab07ee266a7ff235fe7e`
- **Author**: norbu35
- **Size**: +5 / −1, 2 files
- **Closes**: #23982

## Files changed

- `packages/opencode/src/lsp/server.ts` (+4 / −0)
- `packages/opencode/src/lsp/client.ts` (+1 / −1)

## Summary of change

Adds an optional `initializeTimeout?: number` field to `LSPServer.Handle`
(`server.ts` line ~30 in diff). The client (`client.ts` line 279) replaces
the hardcoded `INITIALIZE_TIMEOUT_MS` with
`input.server.initializeTimeout ?? INITIALIZE_TIMEOUT_MS`. JDTLS and
KotlinLS both opt in with `initializeTimeout: 180_000`.

## Specific-line review

- `client.ts:279` — `??` fallback is correct: when the field is `undefined`
  for any other server, behavior is byte-identical to the previous
  hardcoded constant. Good blast radius containment.
- `server.ts` (new field at L30) — the JSDoc `/** Override the default 45 s
  initialize-request timeout (ms). */` documents both the unit and the
  default, which is exactly what a future contributor needs.
- `server.ts:1295` (JDTLS) and `:1395` (KotlinLS) set 180 000 ms = 3 min.
  This is a reasonable cap for "Gradle sync + workspace index" cold starts;
  it is also short enough that a wedged JVM still surfaces a real error
  rather than hanging the agent.

## Things to consider (non-blocking nits)

- The 180 s number is an opinionated default. Consider exposing it via the
  same per-server `initialization` block so power users can override it
  without forking — but that's a follow-up, not a merge blocker.
- No new tests, but the change is purely "swap constant for nullish-coalesced
  field" plus two literal initializers; the existing `test/lsp/` suite
  exercising the init path is sufficient.

## Verdict

**merge-as-is** — minimal, correct, well-scoped fix to a real timeout
regression for JVM-based language servers. Default behavior for every other
server is preserved.
