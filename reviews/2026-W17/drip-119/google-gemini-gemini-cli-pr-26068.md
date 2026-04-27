# PR #26068 — fix(cli): make MCP ping optional in list command and use configured timeout

- **Repo**: google-gemini/gemini-cli
- **PR**: #26068
- **Head SHA**: `6060719b` (latest; original `0f3b00b0` was force-pushed)
- **Author**: cocosheng-g (Coco Sheng)
- **Size**: +88 / -4 across 2 files
- **URL**: https://github.com/google-gemini/gemini-cli/pull/26068
- **Verdict**: **merge-as-is**

## Summary

Two-part fix to the `mcp list` command's connection probe. Closes
two real misshapes:

1. **`ping` was treated as required**: per the MCP spec, `ping` is
   optional and some MCP servers (notably some first-party servers,
   per the inline comment) don't implement it. The previous code
   marked any server that connected successfully but failed
   `ping()` as not-CONNECTED, which was reporting healthy servers
   as broken in the user's `mcp list` output.
2. **Hard-coded 5-second timeout** on `connect()` and no timeout on
   `ping()`. This was ignoring the user-configured per-server
   `timeout` field in the MCP config, and the default 5s was below
   the typical MCP-default of 600s for cold-start servers — so any
   MCP that took >5s to come up was reported as failed even though
   the actual runtime would have waited.

## Specific changes

- `packages/cli/src/commands/mcp/list.ts:20` — adds
  `MCP_DEFAULT_TIMEOUT_MSEC` to the imports from `@google/gemini-cli-core`.
- `packages/cli/src/commands/mcp/list.ts:130-131` — replaces
  `await client.connect(transport, { timeout: 5000 }); // 5s timeout`
  with `const timeout = config.timeout ?? MCP_DEFAULT_TIMEOUT_MSEC;
  await client.connect(transport, { timeout });`. Per-server
  configured `timeout` wins, falls back to the shared default
  (test mock sets it to 600000 ms = 10 min, matching the MCP
  ecosystem default).
- `packages/cli/src/commands/mcp/list.ts:134-144` — wraps
  `await client.ping({ timeout })` in a `try/catch` that demotes a
  ping failure to `debugLogger.debug(...)`. The fall-through
  return value is still `MCPServerStatus.CONNECTED` because connect
  succeeded — exactly the right behavior given the MCP spec's
  optional-ping language. The inline comment "Ping is optional per
  MCP spec - some servers (e.g. Google first-party) don't implement
  it. A successful connect() is sufficient proof of connectivity."
  documents the design choice for the next reader.

## Tests

Three new test cases at `packages/cli/src/commands/mcp/list.test.ts:220-291`:

- `'should display connected status even if ping fails'` (`:220`):
  mocks `client.connect` to resolve, mocks `client.ping` to reject
  with `new Error('Ping failed')`, asserts the debug logger output
  contains `"test-server: /test/server  (stdio) - Connected"`. This
  pins the "ping failure does not demote status" contract.
- `'should use configured timeout for connection'` (`:240`):
  config-supplies `timeout: 12345`, asserts both `client.connect`
  and `client.ping` were called with `{ timeout: 12345 }`. Pins
  the per-server timeout-wiring path on both API surfaces.
- `'should use default timeout for connection when not configured'`
  (`:267`): no per-server timeout, asserts both calls used
  `{ timeout: 600000 }` — the `MCP_DEFAULT_TIMEOUT_MSEC` value
  injected via the vitest mock at `:70`. Pins the fallback path.

## Risks

- **Mock-injected default value**: the test at `:70` mocks
  `MCP_DEFAULT_TIMEOUT_MSEC: 600000`. If the actual upstream constant
  in `@google/gemini-cli-core` is anything other than 600000 ms, the
  test passes but the production behavior diverges from the test
  expectation. A defensive cross-check (e.g. asserting the imported
  constant matches a known value, or referencing the constant
  directly rather than the mocked literal) would close this gap.
  Probably fine in practice since the constant name is canonical and
  unlikely to change silently.
- **Debug-log failure mode**: `debugLogger.debug(...)` of the ping
  failure means in non-debug runs the user has zero feedback that
  `ping` failed. The status correctly stays "Connected" but a future
  bug shape — server connects but is genuinely broken on first real
  request — would be silent. Acceptable trade-off (the ping is
  *optional* and shouldn't surface to users when it fails) but
  worth a doc-level mention that "Connected" is a connect-time
  signal, not a working-tools signal.
- **Try/catch swallow scope**: the catch is around `client.ping(...)`
  only, not the surrounding `client.close()` or status return — so
  any unexpected throw shape from `ping()` (rejection, sync throw,
  abort) all cleanly demote to debug-log + continue. The `e: unknown`
  parameter in the catch (if TS strict) would be the safe shape;
  current `catch (e)` is implicit-any-typed but fine for log-only
  usage.

## Verdict

**merge-as-is**: small surgical fix at the exact two layers that
were broken (timeout source, ping-required treatment), three tests
covering the three new behaviors (ping-fails-but-connected,
configured-timeout-wins, default-timeout-fallback), inline comment
explaining the spec-aligned design choice. Force-pushed to a new
SHA but the diff is unchanged in shape.

## What I learned

The MCP spec's "ping is optional" language is the kind of subtle
protocol detail that lands silently in the bugged shape — a
"connect + verify" probe naturally wants to verify with a ping,
and that ends up reporting healthy-but-non-pinging servers as
failed for the lifetime of that release. The right reading of
"optional" in a connectivity-probe context is "a successful ping is
*confirming* evidence; a failed ping is *not contradicting*
evidence". The `try/catch + debug-log + return CONNECTED` pattern
in this PR encodes that asymmetry correctly. The paired
"configured timeout wins, default falls back" pattern is the
canonical config-precedence shape and the three tests cover all
three outcomes — pingfail-CONNECTED, configured-timeout-wired,
default-timeout-wired — which is the right test matrix size for
this surface.
