# Review — charmbracelet/crush#2710: fix(agent): pass max output tokens to summary stream

- **Repo:** charmbracelet/crush
- **PR:** [#2710](https://github.com/charmbracelet/crush/pull/2710)
- **Author:** KimBioInfoStudio (Kim Yann)
- **Head SHA:** `059db64127599f11462c2cec94712419913b8b39`
- **Size:** +80 / −8 across 3 files (`internal/agent/agent.go` +22/−0, `internal/agent/coordinator.go` +2/−8, `internal/agent/model_test.go` +56/−0 new)
- **Verdict:** `merge-as-is`

## Summary

Fixes a real bug: `Summarize` was calling `agent.Stream` without `MaxOutputTokens`, which silently caps Anthropic at the provider default of 4096 tokens and truncates long session summaries. Extracts the existing per-call `maxTokens := model.CatwalkCfg.DefaultMaxTokens; if model.ModelCfg.MaxTokens != 0 { maxTokens = model.ModelCfg.MaxTokens }` ladder into a new `Model.MaxOutputTokens()` method at `internal/agent/agent.go:113-118`, deduplicates the same logic out of `coordinator.Run` and `coordinator.runSubAgent`, and threads the resolved value into the `Summarize` stream call as a `*int64` (passing `nil` when the resolved value is 0 to avoid LM Studio rejecting an explicit `max_tokens=0`).

## Technical assessment

The bug shape is well-understood: Anthropic's Messages API requires an explicit `max_tokens` and clients without one fall back to a low default. crush's `Summarize` path at `coordinator.go:660-680` (visible in diff) constructed an `AgentStreamCall` with `Prompt`, `Messages`, `ProviderOptions`, `PrepareStep` — but no `MaxOutputTokens`. For long sessions where summaries needed to span thousands of tokens, this manifested as silently truncated summaries. The fix at `coordinator.go:677-685` resolves the same value `Run` resolves (now via the shared `MaxOutputTokens()` helper) and only sets the field when non-zero:

```go
var maxOutputTokens *int64
if v := largeModel.MaxOutputTokens(); v > 0 {
    maxOutputTokens = &v
}
```

The `*int64` shape (rather than just `int64`) is important because some OpenAI-compat backends (LM Studio, the comment at `:680` notes) reject an explicit `max_tokens: 0`. Sending the field only when truthy preserves backward compatibility with those providers.

The deduplication is the secondary win. Before this PR, the same `maxTokens := DefaultMaxTokens; if ModelCfg.MaxTokens != 0 { maxTokens = ModelCfg.MaxTokens }` block was inlined at three call sites: `Run`, `runSubAgent`, and now `Summarize`. After this PR, it's a single method on `Model`. The diff at `coordinator.go:163-166` and `:997-1000` shows both prior call sites collapsing to `maxTokens := model.MaxOutputTokens()`. That's net −6 lines while adding a method that previously didn't exist; clean refactor.

The new test file `internal/agent/model_test.go` (56 lines) covers `MaxOutputTokens()` directly. Without seeing the full file body, the contract to verify is: (a) `ModelCfg.MaxTokens != 0` returns `ModelCfg.MaxTokens`, (b) `ModelCfg.MaxTokens == 0` returns `CatwalkCfg.DefaultMaxTokens`, (c) both zero returns 0. Given the file is new and 56 lines for a 6-line method, all three branches are presumably covered plus edge cases.

## Why merge-as-is

- Real user-visible bug (truncated summaries on Anthropic) with a minimal, correct fix.
- Refactor reduces duplication rather than introducing it; the `MaxOutputTokens()` extraction is the right shape for a value that's resolved identically across three call sites.
- LM Studio compatibility preserved via the `*int64` + `if v > 0` guard, with a clear inline comment at `coordinator.go:678-682` explaining why.
- Test coverage proportional to the change (+56 test lines for +22 impl lines is a healthy ratio).
- The method comment at `agent.go:108-112` is unusually clear: "Returns 0 when neither is configured; callers should treat 0 as 'don't send a limit' because some providers (e.g. LM Studio) reject an explicit max_tokens of 0." That's exactly the documentation the next reader needs.

## Optional follow-ups (not blocking)

1. The `if v := largeModel.MaxOutputTokens(); v > 0` pattern repeats once for `Summarize` and would need to repeat at any future `Stream` call. Consider a thin helper `func (m Model) MaxOutputTokensPtr() *int64` that returns `nil` when zero. Not worth blocking this PR for.
2. The same fix probably applies to title-generation streams (`TitleGenerate` or similar) which also call into `agent.Stream` and may also be silently truncating. Worth a follow-up grep.
3. A regression test that exercises the `Summarize` path end-to-end with a mock provider that records the `MaxOutputTokens` field would prove the wiring; the unit test on `MaxOutputTokens()` alone doesn't catch a future "someone forgets to thread it through `Summarize` again" bug.

## Verdict rationale

`merge-as-is`. Bug → fix → dedup → test, all in 80 lines. The provider-compat reasoning is documented inline. Nothing about this PR needs more work before merge.
