# sst/opencode #25663 — chore: update acp sdk

- SHA: `02a7263971262cf75ee245821feef01bc825e4b7`
- State: OPEN, +37/-6 across 5 files
- Files: `bun.lock`, `packages/opencode/package.json`, `packages/opencode/src/acp/agent.ts`, `packages/opencode/src/acp/session.ts`, `packages/opencode/test/acp/agent-interface.test.ts`

## Summary

Bumps `@agentclientprotocol/sdk` from `0.16.1` to `0.21.0` and adapts the agent to the new SDK surface: adds `closeSession` handler, declares `close: {}` capability, and renames `unstable_resumeSession` → `resumeSession` (capability-gated by the SDK router instead of name-prefix). Also adds `ACPSessionManager.remove()` and updates the interface-compliance test list.

## Notes

- `packages/opencode/src/acp/agent.ts:570` — adding `close: {}` to `sessionCapabilities` is the right SDK 0.21 contract; clients now know the agent supports explicit session close.
- `packages/opencode/src/acp/agent.ts:803` — renaming `unstable_resumeSession` → `resumeSession` looks correct given the test comment "Capability-gated methods checked by the SDK router". Worth double-checking that no external integration was calling the `unstable_` name (no shim left behind).
- `packages/opencode/src/acp/agent.ts:834-851` — `closeSession` removes from the manager, fires `sdk.session.abort` with `throwOnError: true` but then `.catch(...)` swallows the error to a log line. That's defensive but the `throwOnError: true` becomes irrelevant — either drop it or let the error propagate. Minor.
- `packages/opencode/src/acp/agent.ts:850` — `permissionQueues.delete(sessionId)` is correct cleanup; good catch.
- `packages/opencode/src/acp/session.ts:117-121` — `remove()` returns the session before deletion. Tiny race-safety nit: get-then-delete is fine for single-threaded JS, no concern.
- `packages/opencode/test/acp/agent-interface.test.ts:37-41` — moving `resumeSession`/`closeSession` out of the unstable-prefix list and keeping `unstable_forkSession`/`unstable_setSessionModel` in matches the actual SDK 0.21 schema.
- No SDK 0.16→0.21 changelog cross-check in this PR — would be nice to confirm there are no breaking changes to `forkSession` / `setSessionModel` that this PR is silently inheriting.

## Verdict

`merge-after-nits` — clean up the `throwOnError: true` + `.catch` contradiction in `closeSession`, and confirm no external callers depended on the `unstable_resumeSession` name.
