# charmbracelet/crush #2752 — refactor: cache-aligned summarization for compaction requests

- **Head SHA:** `efb4dc1fcd7c7ff5e0f988ceffc9e2aea4f11764`
- **Files:** `internal/agent/agent.go`, `internal/agent/templates/summary.md` (+95/-56)
- **Verdict:** `merge-after-nits`

## Rationale

Real cost optimization, not a cosmetic refactor. The change extracts `buildAgent()` and `buildSystemPrompt()` shared helpers in `agent.go:695+` so both `Run` and `Summarize` produce a byte-for-byte identical agent prefix (system prompt + MCP `<mcp-instructions>` block + tools list with `cache-control` on the trailing tool). Before this PR, `Summarize` called `fantasy.NewAgent(largeModel.Model, fantasy.WithSystemPrompt(string(summaryPrompt)), fantasy.WithUserAgent(userAgent))` (the deleted block at the original `agent.go:653`) with no tools and a different system prompt, which guaranteed a prefix-cache miss on every summarization — and summarization is exactly the call that runs against a long context where cache misses hurt the most. After this PR, `Summarize`'s `PrepareStep` also runs `applyCacheControl` and copies `a.tools.Copy()`, so Anthropic's prompt cache aligns with the main `Run` path.

Nits: (1) the deleted block in `Run`'s `PrepareStep` (the `for i := range prepared.Messages { prepared.Messages[i].ProviderOptions = nil }` loop) was clearing provider options before reapplying cache markers; the new `applyCacheControl` should be audited to confirm it explicitly clears stale `ProviderOptions` on non-cached messages, otherwise a long-running session that toggles cache on/off across runs could leave stale markers (the deleted loop was defensive); (2) `Summarize` now calls `a.tools.Copy()` and includes tools in the summary request — confirm the summarization model has the tool schema appetite for it, and that the `summary.md` template (also touched here) is consistent with passing tools; (3) call out the prefix-cache hit-rate change in the PR description so users on metered Anthropic plans know to expect the cost drop.

Risk is medium: the refactor touches the hot path on every turn (`Run`'s `PrepareStep` ran on every step) and changes what `Summarize` sends to the provider. The fact that it's net `-56` LOC by deduplicating two cache-control implementations is a good sign.
