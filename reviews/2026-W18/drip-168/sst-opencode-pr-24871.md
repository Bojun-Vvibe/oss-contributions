# sst/opencode PR #24871 — fix(mcp): truncate tool names exceeding 64-char provider limit

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/24871
- Head SHA: `7877858011b2d063dcd4f6ddf020402e8d34e0a1`
- Author: georgeglarson
- Size: +98 / −1, 2 files
- Closes: #3523

## Context

OpenAI's tool-use API rejects `tools[].function.name` strings longer than
64 chars with a 400 that opencode previously didn't surface — the failure
mode in #3523 is "TUI silently dies on the next agent turn after enabling
an MCP server with a long name". Anthropic and Google have similar but
slightly different caps (60 / 64 / no cap depending on provider and model
generation), so 64 is the strictest commonly-reachable limit and a
defensible target.

## What the diff actually does

Production change is at `packages/opencode/src/mcp/index.ts:117-135` plus a
single call-site swap at `:666`:

- Adds `MAX_TOOL_NAME = 64` and a `buildToolName(clientName, toolName)`
  helper that returns `sanitize(client) + "_" + sanitize(tool)` unchanged
  when it fits, otherwise truncates to `MAX_TOOL_NAME - 9` chars and
  appends `"_" + Hash.fast(combined).slice(0, 8)`. The 8-char hex hash
  guarantees that two tools sharing the truncated prefix still land on
  distinct keys (collision probability ~2⁻³² per pair, fine for the
  per-server tool-count regime).
- The previous `result[sanitize(clientName) + "_" + sanitize(mcpTool.name)]
  = convertMcpTool(...)` is replaced by `result[buildToolName(clientName,
  mcpTool.name)] = convertMcpTool(...)`.

Test side adds two cases at `packages/opencode/test/mcp/lifecycle.test.ts:786-862`:

- `tools() truncates names exceeding 64 chars and preserves uniqueness`
  uses a 35-char server name plus two tool names that share the first 55
  chars and asserts (a) both register, (b) both are ≤64 chars, (c) `new
  Set(keys).size === 2` (collision check), and (d) every key matches
  `/_[0-9a-f]{8}$/` (hash suffix is present).
- `tools() leaves short names untouched` pins that the truncation path
  doesn't fire on the common case (`short_ping` should land verbatim).

## Risks and gaps

1. **The truncation point is silent to the model**. The `log.warn` lands in
   server logs but the model itself sees a tool name like
   `chrome-devtools-aaa…_perform_extreme_a3f7b21c` that bears no resemblance
   to what its training data or tool spec would suggest. If the model is
   reasoning about tool selection by name (it often is — names are
   load-bearing prompt context), it may now systematically misroute among
   the truncated tools. A description-side hint
   (`"originally <full name>"` prepended to `convertMcpTool`'s description)
   would give the model a recovery path without breaking the wire schema.

2. **Hash is over `combined` (client + tool), not `mcpTool.name` alone**.
   That's correct for collision avoidance within one client but means two
   different clients exposing the same long tool name get different
   suffixes — fine, but worth a one-line comment so the next maintainer
   doesn't "simplify" to hashing just the tool name and accidentally let
   `clientA_<truncated>_<hash>` collide with `clientA_<other_tool>_<same hash>`
   in some pathological case.

3. **64 is hardcoded, not provider-aware**. Anthropic's limit is 64 too,
   Google's is 64 (older), OpenRouter passes through. But if opencode ever
   targets a provider with a 60-char limit (some MCP-bridge SaaS products
   in the wild), the cap silently lets through names that the actual
   provider rejects. A `MAX_TOOL_NAME` lookup keyed on the active provider
   would be more durable, though out of scope for a one-issue fix. At
   minimum, name the constant `OPENAI_MAX_TOOL_NAME` or document the
   provider it tracks.

4. **Sanitization happens before truncation, not after**. `sanitize` maps
   non-`[a-zA-Z0-9_-]` to `_`, which can grow nothing but does mean the
   pre-sanitize length isn't what gets capped. Probably fine because the
   cap is on the post-sanitize length (which is what providers see), but
   worth confirming with a unicode-tool-name test case to make sure a name
   that sanitizes to >64 chars from a <64-char source still lands under
   the cap.

5. **No test for the boundary case (combined.length === 64)**. The current
   tests cover much-shorter and much-longer; a `combined.length === 64`
   case pinning that the no-truncation branch fires (not the hash branch)
   would lock the off-by-one.

6. **`Hash.fast` collision risk is not the main concern, but stability
   across opencode versions is**. If `Hash.fast`'s implementation changes
   in a future release, the same MCP server's tool names will silently
   change between versions, which breaks any persisted state keyed by the
   tool name (recent-tools history, audit logs, prompt caches). A frozen
   hash function (or a comment pinning the dependency) would prevent
   surprise migrations.

## Suggestions

- Prepend `"originally <full name>"` to the truncated tool's description so
  the model can recover the intent.
- Rename `MAX_TOOL_NAME` → `OPENAI_MAX_TOOL_NAME` (or document the provider
  the cap tracks) to make the constraint discoverable.
- Add a boundary-case test (`combined.length === 64`) pinning the
  no-truncation branch.
- Add a unicode/multibyte tool-name test to confirm sanitization-then-truncation
  ordering doesn't surprise.
- Comment on `Hash.fast`'s stability guarantee — if it changes, every
  truncated tool's name changes silently across opencode versions.

## Verdict

**merge-after-nits** — surgical fix that solves the silent-die problem, with
a discriminating collision test that's the right shape. The nits are about
making the model-side surface (description hint) and forward-compat
(provider-aware cap, hash stability) cleaner, not about defects in the
landing diff.
