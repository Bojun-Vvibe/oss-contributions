# google-gemini/gemini-cli #25963 — fix(mcp): expand env vars in stdio args

- **Repo**: google-gemini/gemini-cli
- **PR**: #25963
- **Author**: onthebed
- **Head SHA**: da312a31504686f5f864ef071c411b3413aee31b
- **Size**: +39 / −2 across two files (one prod, one test).

## What it changes

Five-line behavior fix in `packages/core/src/tools/mcp-client.ts:2274-2293`
plus a 30-line regression test.

Before, MCP `stdio` server configs that included `${VAR}`
references in their `args` array (e.g.
`args: ["run", "--env", "DISCORD_TOKEN=${DISCORD_TOKEN}"]`)
were passed through to `StdioClientTransport` literally
— the spawned subprocess saw `${DISCORD_TOKEN}` as a string,
not the env var's value. Pre-existing code in the same
function already expanded env vars in `mcpServerConfig.env`
via `expandEnvVars(... , expansionEnv)` at the surrounding
context, but `args` was not threaded through the same
expansion.

The fix:

```ts
const expandedArgs = (mcpServerConfig.args || []).map((arg) =>
  expandEnvVars(arg, expansionEnv),
);

let transport: Transport = new StdioClientTransport({
  command: mcpServerConfig.command,
  args: expandedArgs,                    // was: mcpServerConfig.args || []
  env: finalEnv,
  cwd: mcpServerConfig.cwd,
  stderr: 'pipe',
});
```

Plus a follow-up bug-fix at `:2290`: the `xcrun mcpbridge`
detection arm now checks `expandedArgs.includes('mcpbridge')`
instead of `mcpServerConfig.args?.includes('mcpbridge')`. If
someone wrote `args: ["${BRIDGE_NAME}"]` with
`BRIDGE_NAME=mcpbridge`, the wrapping `XcodeMcpBridgeFixTransport`
would have been skipped pre-fix.

## Strengths

- **Correctly identifies the symmetry gap.** The `env`
  field was already getting `expandEnvVars` treatment
  upstream in this function (the diff shows
  `expansionEnv` already in scope), but the `args`
  field — which is the conventional place users put
  per-server secrets via `--env KEY=${KEY}` patterns
  passed through to a Docker invocation — was not.
  Fix restores the "all stdio config strings are
  expanded against the same env" invariant.
- **Detection-arm correctness fix is a real bug**, not
  cosmetic. The `xcrun mcpbridge` special-case wrap at
  `:2289-2293` exists because that bridge returns JSON
  in `content` instead of `structuredContent` and needs
  a transport adapter. Pre-fix, any user who templated
  the bridge name (e.g. via env var) silently lost the
  adapter and got malformed responses. Switching the
  predicate to `expandedArgs.includes('mcpbridge')`
  makes the detection robust to the same expansion the
  spawn now uses.
- **Test pins the canonical use case.** The new test at
  `mcp-client.test.ts:2160-2189` writes
  `process.env.DISCORD_TOKEN = 'real-discord-token'`,
  configures
  `args: ['run', '--env', 'DISCORD_TOKEN=${DISCORD_TOKEN}']`,
  and asserts the spawned transport receives
  `['run', '--env', 'DISCORD_TOKEN=real-discord-token']`.
  That's exactly the Docker-MCP-server pattern this
  feature is meant to support, and the assertion uses
  `mockedTransport.mock.calls[0][0]` to reach into the
  actual constructor args rather than asserting on
  intermediate state.
- The `try {} finally { process.env = originalEnv; }`
  scoping is correct — restores the env even if the
  transport setup throws — so the test won't pollute
  sibling tests.

## Concerns / nits

- **No test for the `xcrun mcpbridge` predicate fix.**
  The diff fixes a real bug at `:2290` (templated
  bridge name skipping the adapter wrap) but only the
  Docker-args path is regression-tested. A second
  test setting `args: ['${BRIDGE_NAME}']` with
  `BRIDGE_NAME=mcpbridge` and asserting that the
  resulting `transport instanceof XcodeMcpBridgeFixTransport`
  would lock the second contract.
- **No coverage of the unset-env-var case.** What does
  `expandEnvVars(arg, expansionEnv)` do when the var
  is not in `expansionEnv` — leave the literal
  `${UNSET_VAR}` in place, replace with empty string,
  or throw? The behavior is load-bearing for security
  (an empty-string substitution into
  `--env DISCORD_TOKEN=` would silently send blank
  credentials to a Docker container), and a one-line
  test pinning that decision would prevent future
  refactors from accidentally flipping it.
- **`(mcpServerConfig.args || []).map(...)`** allocates
  a new array even when `args` is `undefined`. Negligible
  overhead, but the pre-fix `mcpServerConfig.args || []`
  was at least consistent in re-using the same empty
  array across call sites — not worth changing for a
  hot path that runs once per MCP server startup.
- The remaining call site near `:2289-2293` was the only
  other place reading `mcpServerConfig.args`; a quick
  `rg "mcpServerConfig\.args"` confirmed it's not
  referenced elsewhere in this file post-fix. Worth
  re-confirming after rebase.

## Verdict

**merge-after-nits.** Fixes a real config-pattern gap
(templated `args` not getting expanded, breaking the
canonical Docker-MCP-server idiom) and a real
correctness side-effect (`xcrun mcpbridge` detection
now sees the expanded value), with a focused
regression test. Want a second test for the
`xcrun mcpbridge` arm and a one-line pin for the
unset-var fallback semantics before merge.

## What I learned

When a config field comes in two parallel forms
(`env: { KEY: VAL }` and `args: [..., "KEY=VAL", ...]`)
and one path runs through a normalization step (env-var
expansion) while the other doesn't, every user who
discovers the working pattern via the documented form
gets bitten when they switch to the equivalent
alternative form. The bug is invisible until a user
notices their secret is literally `${DISCORD_TOKEN}` in
the spawned process's argv. The fix is mechanical, but
the right *design* invariant to extract and document is
"all string-shaped values in this config object go
through `expandEnvVars` before being handed to the
spawn", and a unit test asserting that invariant
across all stringy fields would catch the next gap
before users do.
