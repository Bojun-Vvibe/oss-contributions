---
pr: google-gemini/gemini-cli#25937
sha: 83eb00719d611907dac11f6e2e8d3d4daba9bb7f
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(core): prevent MCP tool wiping on background discovery failures

URL: https://github.com/google-gemini/gemini-cli/pull/25937
Files: `packages/core/src/tools/mcp-client.ts`, `packages/core/src/tools/mcp-client.test.ts`
Diff: 3+/2-

## Context

`discoverTools` in `packages/core/src/tools/mcp-client.ts` is the path that
runs whenever a connected MCP server fires a `tools/list_changed`
notification. Until this PR, when `discoverTools` threw — transient network
blip, server-side error, malformed schema — the catch block at
`mcp-client.ts:1340-1346` logged the error then returned `[]`. The
notification handler upstream then took the empty result as authoritative
and called `removeMcpToolsByServer`, wiping every tool from a previously
healthy server because of one transient failure mid-session.

## What's good

- The fix is a single line at `mcp-client.ts:1347`: `throw error;` after
  the existing `logger.error(...)` call. This propagates the failure out of
  `discoverTools` instead of returning a fake-empty success, which means
  the upstream notification handler's existing try/catch (visible at the
  test's `await notificationCallback();` call site) absorbs the error
  *without* mutating the registry.
- Test inversion at `mcp-client.test.ts:1462-1465`: the prior assertion
  `expect(mockedToolRegistry.removeMcpToolsByServer).toHaveBeenCalled()`
  was literally documenting the bug ("on discovery failure we wipe tools");
  the new assertion `not.toHaveBeenCalled()` documents the fix. Comment is
  updated to match (`"Should NOT try to remove tools when discovery fails"`).
  This is the right pattern for behavior-flip fixes — the test diff is the
  spec change.
- The "should NOT emit success feedback" assertion at `:1468-1470` is
  preserved, so success-feedback semantics aren't accidentally widened.
- Three-line diff, clear blast radius — only the failure path changes,
  the happy path still returns the discovered tool list unchanged.

## Verdict reasoning

Two-line correctness fix backed by a behaviour-flipping test, addresses
a real "transient failure → permanent tool loss until reconnect" bug.
Land it. Only nit-of-nits would be a more descriptive log line at
`:1344-1346` distinguishing transient from permanent failure for
operators, but that's not blocking.
