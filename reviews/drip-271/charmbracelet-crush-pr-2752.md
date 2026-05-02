# charmbracelet/crush #2752 — refactor: cache-aligned summarization for compaction requests

- URL: https://github.com/charmbracelet/crush/pull/2752
- Head SHA: `efb4dc1fcd7c7ff5e0f988ceffc9e2aea4f11764`
- Author: @lloydzhou
- Stats: ~150 net lines refactored, single file `internal/agent/agent.go`

## Summary

Extracts two helpers — `buildSystemPrompt`, `buildAgent`, and
`applyCacheControl` — and routes both `Run` and `Summarize` through
them so the agent prefix (system prompt + tools + cache-control
markers) is byte-for-byte identical between the two paths. Goal: hit
the provider prefix-cache on summarization requests, claimed ~85%
input-token savings per compaction.

## Specific feedback

- `internal/agent/agent.go:179-181, 587-590` — both call-sites now use
  `agent := a.buildAgent()`. The motivating insight (compaction shares
  98% of its prefix with the prior turn) is correct and the fix shape
  is the right one.
- `internal/agent/agent.go:697-720` — `buildSystemPrompt()` correctly
  consolidates the system-prompt + MCP-instructions concatenation that
  was previously duplicated. Note: this reads `a.systemPrompt.Get()`
  and `mcp.GetStates()` at call time on every `buildAgent()`, so a
  concurrent `SetSystemPrompt` between `Run` and `Summarize` could
  *break* cache alignment (different prefix → cache miss). That's
  arguably acceptable behaviour, but worth a code comment so a
  future maintainer doesn't try to "optimize" by snapshotting once.
- `internal/agent/agent.go:722-743` — `buildAgent()` snapshots
  `a.tools.Copy()` and stamps cache-control on the last tool. Since
  it's called once per `Run`/`Summarize` and tools can mutate via
  `SetTools` between those calls, alignment is **not** strictly
  guaranteed across long-lived sessions. The PR description of "85%
  savings" assumes tools haven't changed — a worst-case (MCP server
  reconnects between turn N and the compaction trigger) silently
  degrades to no caching. Consider logging a metric when the
  pre-compaction tool set hash differs from the prior turn's so users
  can debug cache misses.
- `internal/agent/agent.go:746-772` — `applyCacheControl` collapses two
  previously-separate marker passes into one. The early `for i := range
  messages` clears stale `ProviderOptions` first, which is critical:
  before this PR, `Summarize` skipped the clear (the `for i :=
  range prepared.Messages { prepared.Messages[i].ProviderOptions = nil }`
  loop only existed in `Run`'s PrepareStep). Good catch.
- `applyCacheControl:761-765` — the "last system message + last 2
  messages" marker logic is preserved verbatim. Correct semantics.
- `internal/agent/agent.go:629-633` — `Summarize`'s PrepareStep now
  also assigns `prepared.Tools = a.tools.Copy()`. This is the
  load-bearing change: previously the summarizer ran tool-less, so
  the summarization request prefix didn't include the tool definitions
  → no chance of prefix-cache hit. Now it does. Worth a regression
  test asserting the request body includes the tools array.
- No new tests. For a refactor that claims a measurable performance
  win (~85% token savings), at minimum an assertion that
  `buildAgent()` returns equal-shaped prefixes from `Run` and
  `Summarize` would lock in the invariant.
- Comment block at lines 736-743 ("Cache-Aligned Summarization")
  earns its keep — that paragraph IS the PR description and should
  stay in the source.

## Verdict

`merge-after-nits` — solid refactor with a real perf upside. Add a
test fixing the alignment invariant and a code comment about the
SetTools-between-turns degradation; the rest is shippable.
