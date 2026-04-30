# google-gemini/gemini-cli PR #26247 — fix: expand template vars in MCP stdio config

- PR: https://github.com/google-gemini/gemini-cli/pull/26247
- Head SHA: `00a8540307657acec5d56f331709d039e3743ba4`
- Files touched: `packages/core/src/tools/mcp-client.ts`, `packages/core/src/tools/mcp-client.test.ts` (+67/-4, 2 files)

## Specific citations

- Bug: `createTransport` at `mcp-client.ts:~2280` was previously expanding `${VAR}`/`$VAR` (`expandEnvVars`) only on the `env` map values. The `command`, `args`, and `cwd` fields of an MCP stdio server config were passed *raw* into `new StdioClientTransport({...})` (`:2295-2299` pre-fix) — meaning a config like `{"command": "{{HOME}}/.bun/bin/bun"}` would `spawn` a literal path containing the `{{HOME}}` placeholder and fail with ENOENT. Same bug for `args` array entries and `cwd`.
- Fix introduces a single helper `expandMcpServerConfigValue(value: string, env: Record<string, string | undefined>): string` at `mcp-client.ts:1168-1178` that runs `value.replace(/\{\{(\w+)\}\}/g, (match, name) => env[name] ?? match)` *first* (the `{{VAR}}` brace form) then pipes through the existing `expandEnvVars(..., env)` (the `${VAR}`/`$VAR` shell form). The fallback `?? match` preserves the literal placeholder if the var is unset — surfaces the misconfiguration via spawn failure rather than silently substituting empty string.
- Three call sites at `:2293-2305` now expand all three fields:
  - `const command = expandMcpServerConfigValue(mcpServerConfig.command, expansionEnv)`
  - `const args = mcpServerConfig.args?.map((arg) => expandMcpServerConfigValue(arg, expansionEnv))`
  - `const cwd = mcpServerConfig.cwd ? expandMcpServerConfigValue(mcpServerConfig.cwd, expansionEnv) : undefined`
- The `env`-map expansion at `:2284-2287` is updated to use the same helper for symmetry: `finalEnv[key] = expandMcpServerConfigValue(value, expansionEnv)` — previously called `expandEnvVars(value, expansionEnv)` directly. This means `env` values now also support the `{{VAR}}` brace form, which is a small behavior expansion (additive, backward-compat).
- Regression at `mcp-client.test.ts:1996-2033`: stubs `process.env.HOME = "/home/test-user"` and asserts the spawn args at `:2025-2031` carry the *expanded* paths — `command: "/home/test-user/.bun/bin/bun"`, `args: ["/home/test-user/server.js", "/home/test-user/config.json"]` (note: the second arg uses the `$HOME` shell form, exercising both expansion paths in one test), `cwd: "/home/test-user/workspace"`, `env: expect.objectContaining({SERVER_ROOT: "/home/test-user/mcp"})`.

## Verdict: merge-as-is

## Concerns / nits

1. **Clean, surgical fix** for a real config-time UX bug. The two-stage expansion (`{{VAR}}` then `${VAR}`/`$VAR`) is the right shape because the `{{VAR}}` brace form is the documented MCP config style (per the issue body) while `${VAR}`/`$VAR` is the implicit shell form most users reach for. Supporting both with deterministic order means the same config works regardless of which convention the user picked.
2. **Missing-var fallback `?? match` at `:1174`** is the right call — preserves the literal `{{HOME}}` so the spawn fails loudly with a path-not-found error rather than silently expanding to an empty string and spawning the binary at `/`. This is the correct fail-loud-on-misconfiguration policy. Worth a one-line comment naming the intent so a future "improvement" doesn't switch to `?? ""`.
3. **`expansionEnv` reaches into `process.env` plus the explicit MCP-config `env` map** — the priority order is implicit in the surrounding code (not visible in this diff slice). Worth confirming a test case where the same key is set in both `process.env` and `mcpServerConfig.env`: which wins? Documented behavior either way prevents drift.
4. **Single test case covers the happy path only**. Add three negative cases: (a) unset var → literal `{{FOO}}` survives in spawn args, (b) `args` is `undefined` → `args || []` at `:2306` still works, (c) `cwd` is `undefined` → no expansion, no spawn-arg key. These are one-line `expect` calls each and lock the edge cases the fallback `?? match` was written for.
5. **The regex `/\{\{(\w+)\}\}/g` at `:1173`** matches `\w` only — so `{{MY_VAR}}` works but `{{MY-VAR}}` does not, and `{{ MY_VAR }}` (with whitespace) does not. MCP config conventions in the wild include kebab-case and whitespace-tolerant forms; not a blocker but worth a config-doc note that the brace form is `\w`-only. The existing `expandEnvVars` may or may not be looser — confirm parity.
6. **`stderr: 'pipe'` literal at `:2310`** is preserved from the pre-fix shape; no concern, just noting the spawn options surface is otherwise unchanged.
7. **`process.env = originalEnv` cleanup in `try/finally` at `:2002, 2032`** is the right pattern, but the test mutates a *shared* `process.env` object — under parallel test execution (vitest's default) other tests reading `process.env.HOME` during this test's window would see the stub. Vitest's per-file isolation may save this, but consider using a scoped `vi.stubEnv('HOME', '/home/test-user')` instead, which auto-resets on `afterEach`.
