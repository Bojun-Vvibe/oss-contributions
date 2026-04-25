# crush #2710 — pass max output tokens to summary stream

- URL: https://github.com/charmbracelet/crush/pull/2710
- Head SHA: `059db64127599f11462c2cec94712419913b8b39`
- Verdict: **merge-as-is**

## What it does

`SessionAgent.Summarize()` was streaming through the `fantasy` agent
without setting `MaxOutputTokens` on the `AgentStreamCall`. The fantasy
Anthropic provider hardcodes a 4096-token fallback when callers don't
specify, so long-context sessions silently produced summaries truncated
at 4096 tokens — losing arbitrary trailing content.

The PR factors out the existing token-resolution rule from `coordinator.go`
into a `Model.MaxOutputTokens()` method and uses it in three places:
`coordinator.Run`, `coordinator.runSubAgent`, and the previously-broken
`SessionAgent.Summarize`.

## Diff notes

- New helper at `internal/agent/agent.go`:
  ```go
  func (m Model) MaxOutputTokens() int64 {
      if m.ModelCfg.MaxTokens != 0 {
          return m.ModelCfg.MaxTokens
      }
      return m.CatwalkCfg.DefaultMaxTokens
  }
  ```
  with the comment explicitly calling out that `0` means "don't send a
  limit" because LM Studio rejects an explicit `max_tokens=0`.
- Summarize call site does the safe nil-pointer dance correctly:
  ```go
  var maxOutputTokens *int64
  if v := largeModel.MaxOutputTokens(); v > 0 {
      maxOutputTokens = &v
  }
  ```
  — only sends the field when it's a real positive limit, leaving
  fantasy/Anthropic to use whatever they consider sensible when neither
  user nor catwalk default exists.
- `coordinator.Run` and `coordinator.runSubAgent` both lose the inlined
  if/else and call `model.MaxOutputTokens()` instead. Pure de-duplication.
- New `model_test.go` covers the four cases: ModelCfg set, ModelCfg zero
  with Catwalk default, both zero, and the precedence order.

## Risk surface

- Anthropic users on long sessions will now get the larger Catwalk default
  (or their configured `max_tokens`) instead of the silent 4096 cap. That
  means slightly higher token spend on summaries, but that's the intended
  behavior — the previous behavior was a correctness bug, not a cost knob.
- The `*int64` nil-vs-zero distinction matches what fantasy expects, and
  the inline comment locks in why `0` and `nil` aren't interchangeable.

## Why this verdict

Real bug, surgical fix, the refactor (pulling the token-resolution rule
into one method) is genuinely warranted because three call sites were
duplicating the same if/else. Tests cover the precedence rules. No nits.
