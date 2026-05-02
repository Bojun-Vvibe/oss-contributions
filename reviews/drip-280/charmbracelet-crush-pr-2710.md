# Review: charmbracelet/crush #2710 — fix(agent): pass max output tokens to summary stream

- **PR**: https://github.com/charmbracelet/crush/pull/2710
- **Head SHA**: `059db64127599f11462c2cec94712419913b8b39`
- **Diff size**: +80 / 0 lines, 3 files (`internal/agent/agent.go`, `internal/agent/coordinator.go`,
  new `internal/agent/model_test.go`)

## What it changes

Fixes a silent 4096-token truncation of conversation summaries on Anthropic providers.

1. **New helper** `Model.MaxOutputTokens() int64` (`agent/agent.go:114-119`) that resolves
   the limit as `ModelCfg.MaxTokens > 0 ? ModelCfg.MaxTokens : CatwalkCfg.DefaultMaxTokens`.
   Returns 0 when neither is configured; comment correctly notes that callers should
   treat 0 as "don't send a limit" because LM Studio rejects an explicit `max_tokens: 0`.
2. **`Summarize()` fix** (`agent/agent.go:675-688`): previously did not set
   `MaxOutputTokens` on the `fantasy.AgentStreamCall`, so the upstream
   `fantasy/providers/anthropic/anthropic.go` hardcoded fallback `if params.MaxTokens == 0
   { params.MaxTokens = 4096 }` truncated long summaries on Anthropic but not on OpenAI
   (which passes nil through to the server's larger default — explaining why the bug was
   Anthropic-specific).
3. **Coordinator de-duplication** (`coordinator.go:163`, `:997`): two inline copies of
   the `ModelCfg.MaxTokens` / `CatwalkCfg.DefaultMaxTokens` fallback removed, both now
   call `model.MaxOutputTokens()`. Net 6 duplicated lines gone.
4. **New `model_test.go`**: 4 table-driven cases — `ModelCfg` override wins, catwalk
   default fallback, both zero, explicit-zero-in-`ModelCfg` uses catwalk default.

## Assessment

This is a clean, well-targeted fix with a clear root-cause analysis. The PR body
correctly identifies the asymmetry between Anthropic and OpenAI providers in `fantasy`,
which explains why the bug survived in tree (OpenAI users never hit it).

The "preserve LM Studio compat" carve-out at `agent.go:683-686` is the right call:

```go
var maxOutputTokens *int64
if v := largeModel.MaxOutputTokens(); v > 0 {
    maxOutputTokens = &v
}
```

This matches the existing pattern in `Run` (referenced in the PR body) and avoids sending
`max_tokens: 0` which LM Studio and some OpenAI-compat endpoints reject.

The test cases at `model_test.go:88-122` cover the right matrix. The "explicit zero in
ModelCfg does not override catwalk default" case is the subtle one — it pins the
semantics that `ModelCfg.MaxTokens == 0` means "unset, fall back" rather than "I want
zero tokens." That's the right semantics but worth pinning.

Concerns (minor):

- The fix is applied only to `Summarize()`. The PR body asserts `Run` already had this
  via `coordinator.go:163` — confirmed in the diff. But are there other call sites that
  call `agent.Stream` directly with a missing `MaxOutputTokens`? A `grep -r "AgentStreamCall{"`
  result in the PR body would close this, since the fix is structurally about
  "everywhere that builds an `AgentStreamCall`."
- The `Model.MaxOutputTokens()` method is on the `Model` value receiver — fine for a
  pure resolver, but if `Model` ever grows mutable state this could be a footgun. Not
  blocking.
- `model_test.go:97` uses `t.Parallel()` at both the outer and inner test — good practice.

## Verdict

`merge-as-is` — root cause is correct, fix is minimal and targeted, test coverage
matches the matrix. The de-duplication in `coordinator.go` is a free bonus. Optional:
add a one-line follow-up to grep for any remaining `AgentStreamCall{` construction
sites that might still be missing `MaxOutputTokens`.
