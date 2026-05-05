# sst/opencode #25920 — fix(mcp): support native windows shell execution for local servers

- URL: https://github.com/sst/opencode/pull/25920
- Head SHA: `fa38b038ff7b1d3e758861221c2cac79a2984913`
- Diff: +8 / -2, single file `packages/opencode/src/mcp/index.ts`

## Findings

1. Core change at `packages/opencode/src/mcp/index.ts:386-410` (in `spawnLocalServer` / equivalent): destructures original `mcp.command` into `[baseCmd, ...baseArgs]`, then on Windows substitutes `cmd.exe /c <baseCmd> <baseArgs...>` while leaving non-Windows behaviour identical. The shape of the wrap (`["/c", baseCmd, ...baseArgs]`) is the standard idiom for `cmd.exe`, and it's what `StdioClientTransport` needs because Node's `child_process.spawn` on Windows does **not** resolve `.cmd`/`.bat` shims unless run through a shell. Solid fix for the reported class of bugs (#25904, #22310, #6994).
2. The `BUN_BE_BUN: "1"` env-var injection at line 401 of the post-patch file correctly switched to checking `baseCmd === "opencode"` instead of the new `cmd` (which is now `cmd.exe` on Windows). Without this guard fix, Windows would never set `BUN_BE_BUN` for `opencode`-as-MCP recursion. Good catch.
3. **Concern — argument quoting**: when `baseArgs` contains spaces, ampersands, carets, pipes, or `&&`, `cmd.exe /c <prog> <arg with spaces>` will mis-parse them because `cmd.exe` re-tokenizes the tail. `StdioClientTransport` passes args as an array, but Node on Windows ultimately serializes them into a single command line, and `cmd.exe`'s `/c` parsing is notoriously fragile (see Node's `windowsVerbatimArguments` discussion). For a user MCP config like `{"command": ["node", "C:\\Program Files\\my-mcp\\server.js"]}`, the path with spaces will break. Recommend either (a) wrapping each arg in `cmd.exe`-safe quoting before composing the array, or (b) setting `windowsVerbatimArguments: true` on the spawn options if the transport exposes it. At minimum, a regression test with a spaced path should be added.
4. The Spanish-language comment "Inyección nativa del shell para evitar errores de entorno en Windows" is fine as-is but the rest of the codebase uses English comments — translate for consistency.
5. No tests added. Given this touches platform-conditional spawning, a unit test that mocks `process.platform` and asserts the constructed command/args would catch quoting regressions cheaply.

## Verdict

**Verdict:** request-changes
