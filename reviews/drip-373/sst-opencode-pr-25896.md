# sst/opencode #25896 — fix(mcp): support native windows shell execution for local servers

- **Head:** `fa38b03` (`fa38b038ff7b1d3e758861221c2cac79a2984913`)
- **Repo:** sst/opencode
- **Scope:** `packages/opencode/src/mcp/index.ts:386-407` (single file, ~10 lines net)

## What the diff does

Changes the local-MCP `StdioClientTransport` spawn at `mcp/index.ts:389-394` so that on Windows, the user-supplied `mcp.command` array is wrapped in `cmd.exe /c <cmd> <args...>` instead of being exec'd directly. Non-Windows paths are unchanged. The `BUN_BE_BUN` env-injection check at `:404` is correctly migrated to compare against `baseCmd` (the user-intended binary) rather than the now-shimmed `cmd` value, preserving the previous behavior for `command: ["opencode", ...]`.

## Why it is needed

`StdioClientTransport` (Node `child_process.spawn` underneath) on Windows cannot execute `.cmd` / `.bat` shims (npm, npx, uvx, bunx, deno, most package-manager-installed CLIs) without `shell: true` or an explicit `cmd.exe /c`. The current code path silently fails for the common MCP install pattern (`npx -y @modelcontextprotocol/server-foo`) on Windows because `npx.cmd` is not directly executable as a PE binary. This PR is the established Node fix.

## Concerns

1. **`cmd.exe /c` arg-quoting hazard at `:391`.** Splatting `[baseCmd, ...baseArgs]` straight into the `args` array of a `spawn("cmd.exe", ...)` call is only safe if the consumer doesn't apply `windowsVerbatimArguments`-style joining. Node's default behavior on Windows wraps each arg in double-quotes and escapes embedded quotes, which is correct for most cases but breaks for args containing `&`, `|`, `<`, `>`, `^`, `%` — `cmd.exe` will still interpret those after dequoting. If users are passing MCP servers with `&` in args (rare but possible for filter expressions), they will see new injection-shaped failures. Worth either documenting the limitation or using `shell: true` (which has the same issue but at least is the documented Node escape hatch).
2. **Spanish comment at `:391`.** `// Inyección nativa del shell para evitar errores de entorno en Windows` — the rest of the codebase comments in English; this one will trip the next reviewer. Trivial nit.
3. **No test added.** The existing MCP test surface is hard to exercise on Windows in CI, but at minimum a unit test pinning the `args` shape on `process.platform === "win32"` would prevent regression.
4. **PowerShell vs cmd choice.** `cmd.exe /c` is the right pick for compatibility (it's the lowest common denominator and present on all Windows since NT) — explicitly worth calling out in the commit message that this is intentionally not `powershell.exe -Command`.

## Verdict

**merge-after-nits** — translate the Spanish comment to English and (optionally) add a one-line note in the commit body explaining the `cmd.exe` vs `pwsh` choice. The functional change is the canonical Node-on-Windows fix and unblocks a real class of `npx`-launched MCP servers; head `fa38b03`.
