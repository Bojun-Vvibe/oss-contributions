# google-gemini/gemini-cli PR #26457 — fix(cli): improve mcp list UX in untrusted folders

- Repo: `google-gemini/gemini-cli`
- PR: #26457
- Head SHA: `3bb1315b870644dec410773cdf47e2a9c779362e`
- Author: `Adib234`
- Updated: 2026-05-04T19:37:08Z
- Verdict: **merge-after-nits**

## What it does

Tightens the `gemini mcp list` UX when the current folder is untrusted — instead of showing each configured stdio MCP server as `Disconnected` (which made it look like a transport bug), the command now shows them as `Disabled` and prints a one-line warning explaining that the disable is policy-driven, not a connection failure.

Test-only delta visible in this hunk; the production behavior change is implied by the test rewrites. The new test contract is:

- `packages/cli/src/commands/mcp/list.test.ts:365-394` — renamed `should show stdio servers as disconnected in untrusted folders` → `should show stdio servers as disabled in untrusted folders`. The new assertions:
  ```
  expect(debugLogger.log).toHaveBeenCalledWith(
    expect.stringContaining(
      'Warning: MCP servers are configured but disabled because this folder is untrusted.',
    ),
  );
  expect(debugLogger.log).toHaveBeenCalledWith(
    expect.stringContaining('test-server: /test/server  (stdio) - Disabled'),
  );
  ```
- The previous test used `mockedCreateTransport.mockRejectedValue(new Error('Folder not trusted'))` to simulate the transport failure; that mock is removed, implying the new code path no longer attempts `createTransport` at all when `isTrusted: false`.

The PR also rewrites the test fixtures to use a new `createMockSettings({ ... })` helper from `../../test-utils/settings.js` instead of hand-rolling `mergeSettings({}, {}, {}, {}, true)` + spreading defaults — replaces 8 occurrences of the same boilerplate (lines 137, 144-159, 195-201, 217-222, 236-241, 259-264, 282-287, 316-322).

## Strengths

- **Right UX call.** "Disconnected" is the wrong word for "we deliberately chose not to start this." Users debugging "why isn't my MCP server working" in a fresh untrusted folder were chasing a phantom transport problem. The `Disabled` label plus the explanatory warning collapses the support load.
- **Removes `createTransport` call entirely on untrusted.** Eliminates the previous fake "we tried and failed" pattern (`mockRejectedValue(new Error('Folder not trusted'))`) which means real users no longer spawn a doomed connect attempt per server on every `mcp list` invocation in untrusted dirs — small perf win, more honest semantics.
- **`createMockSettings` consolidation is a real improvement.** The old `mergeSettings({}, {}, {}, {}, true)` + spread pattern was repeated 8 times with a `setRemoteAdminSettings` escape hatch grafted on for the admin-allowlist test. The new helper centralizes the "give me a default LoadedSettings with these mcp servers" intent and the admin-controls test (lines 315-355) explicitly uses `setRemoteAdminSettings(adminControls as unknown as AdminControlsSettings)` which is now the documented way to inject admin overrides.
- **Admin-allowlist test rewrite is more accurate.** Old test mutated `settings.admin = { ... mcp: { config: { 'allowed-server': {...} }, requiredConfig: {} } }` directly. New test uses the actual `AdminControlsSettings` shape (`strictModeDisabled`, `mcpSetting.mcpEnabled`, `mcpSetting.mcpConfig.mcpServers`) which matches what `loadSettings` actually produces.

## Nits / concerns

1. **Production-code diff not visible in the hunk above.** The test changes imply that `listMcpServers` now branches on `isTrusted` *before* iterating servers and emits the bulk warning once, but the implementation file (`packages/cli/src/commands/mcp/list.ts`) isn't shown. Verify that the warning is emitted exactly once per invocation (not once per server) — repeating it 12 times for a user with 12 configured servers would be a regression in a different direction.

2. **`Disabled` label conflicts with the existing "disabled via enablement manager" code path.** The last test in the file (lines 408-on, partially visible) is `should display disabled status for servers disabled via enablement manager` — both code paths now print `Disabled`, which means a user can't distinguish "disabled because untrusted folder" from "disabled because `mcpServerEnablement: false`" by reading the per-server line alone. The bulk warning at the top mitigates this, but consider distinct labels (`Disabled (untrusted)` vs `Disabled (config)`) on the per-server line, or at minimum a footer summary.

3. **`mcpRejectedValue` mock removal means the previous test's *transport-error* contract is now unenforced.** If a real transport-rejection bug appears later (DNS, permissions on the unix socket), there's no test asserting that the per-server line shows `Disconnected` with the underlying error reason. Worth keeping a separate test that explicitly stubs a transport failure on a *trusted* folder to lock the `Disconnected` branch.

4. **`adminControls as unknown as AdminControlsSettings`** double-cast at line 339 is a smell — if `AdminControlsSettings` is the intended shape, the test fixture should be typed against it directly so a future field rename in the core type breaks the test. Use a `satisfies AdminControlsSettings` clause instead.

5. **No PR description visible in this view, but the title** ("improve mcp list UX in untrusted folders") matches the diff scope precisely; rare and welcome.

## Verdict
**merge-after-nits** — correct UX call, the per-server label change plus the bulk warning is the right combination, and the test-fixture consolidation is a net positive. Confirm the bulk warning is emitted exactly once (not per-server), consider a distinguishing label between the two `Disabled` paths, and keep a separate `Disconnected` test for the trusted-folder transport-failure branch.
