# google-gemini/gemini-cli PR #25975 — fix(core): expand env vars in mcp server args

- Link: https://github.com/google-gemini/gemini-cli/pull/25975
- Head SHA: `969af149396cffbcc425497b8ca54ebef39ee42e`
- Size: +137 / -10 across 2 files

## Summary

Extends `createTransport` in `mcp-client.ts` (`:2270-2295`) so that environment-variable references like `$VAR` and `${VAR}` are now expanded not only in `mcpServerConfig.env[*]` (existing) but also in `mcpServerConfig.command`, `mcpServerConfig.args[*]`, and `mcpServerConfig.cwd` before being handed to `StdioClientTransport`. Also switches the `env` value expansion source from `expansionEnv` to `sanitizedEnv` so the same allowlist gating that protects env values now applies uniformly across all four expansion sites. Adds three new tests covering args, command-and-cwd, and sequential `${VAR1}_${VAR2}` interpolation.

## Specific-line citations

- `mcp-client.ts:2273`: `expandEnvVars(value, sanitizedEnv)` (was `expandEnvVars(value, expansionEnv)`). This is a meaningful semantic change beyond the headline fix: prior code expanded env values against the un-sanitized expansion environment, meaning a server config could reference an env var that wasn't on the allowlist and the expansion would silently succeed. Switching to `sanitizedEnv` enforces the allowlist as the single source of truth for "what an MCP config can read from the parent process environment", which is the correct security posture.
- `mcp-client.ts:2278-2287`: three new `expandedCommand` / `expandedArgs` / `expandedCwd` locals. `expandedCwd` correctly uses `mcpServerConfig.cwd ? expandEnvVars(...) : undefined` so the `undefined`-cwd inheritance behavior is preserved. `expandedArgs` defaults `args || []` before mapping, matching the prior shape.
- `mcp-client.ts:2289-2294`: `StdioClientTransport({ command: expandedCommand, args: expandedArgs, env: finalEnv, cwd: expandedCwd, stderr: 'pipe' })` — the wiring is mechanical. No behavior change for configs that don't use `$VAR` syntax (idempotent expansion).
- `mcp-client.test.ts:2131-2169`: the *existing* test is migrated from `process.env = {…}` mutation to `vi.stubEnv` + `localContext.sanitizationConfig.allowedEnvironmentVariables = ['GEMINI_TEST_VAR']`. This isn't cosmetic: the prior test passed because `expansionEnv` happened to contain the var, but the new code requires the var to be on the allowlist, so the test setup had to be tightened to match the new contract. Worth calling out because it documents the security tightening.
- `mcp-client.test.ts:2172-2244`: new "expand environment variables in args" pins `--arg1 $GEMINI_TEST_VAR` and `--arg2=value-$GEMINI_TEST_VAR` shapes.
- `mcp-client.test.ts:2207-2243`: "expand in command and cwd" pins `$TEST_HOME/bin/server` → `/home/user/bin/server` and `$TEST_CWD/src` → `/path/to/project/src`.
- `mcp-client.test.ts:2246-2270`: sequential expansion pins `$VAR1-$VAR2` and `${VAR1}_${VAR2}` shapes — both syntactic forms are exercised, which guards against `expandEnvVars` regressing on either dialect.

## Verdict

**merge-after-nits**

## Rationale

The headline fix is correct and useful: `$VAR` in args is a common ergonomic ask for MCP server configs (path to plugin dir, ports, secrets-via-env-name) and the implementation is symmetric with the already-supported env-value expansion. The opportunistic switch from `expansionEnv` to `sanitizedEnv` for env values is the *correct* security tightening — without it, the allowlist had a quiet hole — and the test rewrite at `:2131-2169` honestly surfaces that change rather than hiding it.

Three nits worth addressing: (1) the security-tightening side-effect (env values now allowlist-gated) deserves an explicit note in the PR description and ideally a CHANGELOG line, because users who relied on the prior un-allowlisted behavior will see configs silently start producing literal `$FOO` strings instead of expanded values; (2) the new tests don't cover the *negative* case "var not on allowlist → arg is left as literal `$VAR` (or empty, depending on `expandEnvVars` semantics)" which is the most likely user-visible regression vector; (3) `expandedArgs` doesn't normalize `null`/`undefined` array entries — if a config accidentally has `args: ['--flag', null, '$X']` the `.map` will pass `null` to `expandEnvVars` and break.

