# google-gemini/gemini-cli#26136 — fix(core): disconnect extension-backed MCP clients in stopExtension

- **Repo:** [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)
- **PR:** [#26136](https://github.com/google-gemini/gemini-cli/pull/26136)
- **Head SHA:** `37479f935f71293fde253e9b8ca61911a0e8deda`
- **Size:** +26 / -1 across 2 files
- **State:** OPEN
- **Fixes:** #24050

## Context

`McpClientManager.stopExtension()` was calling `disconnectClient(name, true)`
where `name` is the *MCP server name from the extension config* (e.g.,
`"test-server"`), not the **client key** that the manager actually uses to
index its connection map. The result: extension-backed MCP clients stayed
connected after the extension was unloaded, which leaks subprocess + socket
resources and (worse) means the client keeps appearing in tool listings
under the now-removed extension's namespace.

## Design analysis

The fix is a single-line change at `mcp-client-manager.ts:226-227`:

```ts
// before
return this.disconnectClient(name, true);

// after
const clientKey = this.getClientKey(name, config);
return this.disconnectClient(clientKey, true);
```

This matches the indexing the manager uses everywhere else: clients are
stored under `getClientKey(name, config)`, where the key encodes both the
server name and (presumably) the extension scope from `config`. The previous
code was passing the un-keyed `name`, which would only ever match
non-extension MCP server connections — extension-scoped ones (which use the
fully-qualified key) never matched, so `disconnectClient` was a no-op for
them.

The test (`mcp-client-manager.test.ts:729-752`) is the right shape:

```ts
it('should disconnect extension-backed MCP clients when stopExtension is called (#24050)', async () => {
  const manager = setupManager(new McpClientManager('0.0.1', mockConfig));
  const mcpServers = { 'test-server': { command: 'node', args: ['server.js'] } };
  const extension: GeminiCLIExtension = { name: 'test-extension', mcpServers, isActive: true, ... };

  await manager.startExtension(extension);
  expect(manager.getMcpServerCount()).toBe(1);

  await manager.stopExtension(extension);

  expect(manager.getMcpServerCount()).toBe(0);
  expect(mockedMcpClient.disconnect).toHaveBeenCalled();
});
```

Three things this test gets right:

1. **Round-trip via `startExtension`/`stopExtension`.** The test exercises
   the same `getClientKey` derivation on both ends — start writes under
   the key, stop must read from it. Doesn't hardcode the key shape, so the
   test survives any internal refactor of `getClientKey`.
2. **Asserts on `getMcpServerCount()` returning 0.** That's the
   user-observable contract — "after stopExtension, the count is back to
   what it was before startExtension." Stronger than just asserting
   `disconnect` was called (which the test also does, as a belt-and-braces
   check).
3. **References the issue number in the test name** (`(#24050)`), so the
   regression marker is searchable from a code grep alone.

## Risks / nits

1. **`getClientKey(name, config)` reads `config` but the production code
   reads `mcpServerConfig` from the surrounding `entries` iteration.** The
   diff snippet shows `Object.entries(extension.mcpServers)` mapping into
   the disconnect call — `config` here is the per-server MCP config. If
   `getClientKey` ever changes to require the extension name (e.g.,
   `getClientKey(name, config, extension.name)`), this call site needs to
   be updated. Not a blocker today, but worth a one-line comment that the
   key derivation here must match `connectClient` / `startExtension`'s
   exactly.
2. **No regression test for the non-extension path.** The test pins the
   extension-backed case but not "regular MCP servers (not extension-backed)
   are unaffected by this change." A second test case calling
   `stopExtension` on an extension that *doesn't* own a particular client,
   asserting that client stays connected, would lock the negative
   contract.
3. **`blockedMcpServers` array splicing at line 222 still uses `name`,
   not `clientKey`.** Reading the surrounding context: the `blockedMcpServers`
   list is presumably indexed by raw server name (not by client key), so
   this is correct as written — but a comment distinguishing why `name`
   is right for the blocked-list and `clientKey` is right for the
   disconnect would prevent future copy-paste mistakes.

## Verdict

**merge-as-is.** Single-line correctness fix with a clean
round-trip test that pins the user-observable contract
(`getMcpServerCount` returns 0). The nits are documentation / coverage
extensions, not blockers — the change is too small and well-targeted to
sit on.

## What I learned

- When a manager indexes its own state under a derived key, every mutation
  path (start, stop, connect, disconnect, refresh) must derive the *same*
  key from the *same* inputs. A "derive once, then pass the key" helper
  function called from every site would prevent this whole bug class —
  worth considering as a follow-up.
- Round-trip tests (`start → assert count === N; stop → assert count === 0`)
  are stronger than mock-call assertions because they survive internal
  refactors of the keying scheme. Mock assertions are a useful belt-and-
  braces check but shouldn't be the *primary* assertion.
- Embedding the upstream issue number in the test name (here: `(#24050)`)
  makes the regression marker findable via `grep -r "#24050"` six months
  later, when the bug context has otherwise been forgotten.
