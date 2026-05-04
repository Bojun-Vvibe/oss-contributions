# charmbracelet/crush #2579 — feat(tool): add `ask-user-questions` tool

- **Head SHA:** `c6ee6f7b16a6a68620fc5d802f6ede9d4dc16eed`
- **Size:** +1014 / -5 across many files
- **Verdict:** **merge-after-nits**

## Summary
Adds a new agent tool `ask_user_questions` that lets the model interactively
ask the user multiple-choice questions during a turn. Introduces a new
`internal/questions` service (`Service.Ask`, request/response types, pubsub
broker), wires it through `App.New` (`internal/app/app.go:97`) and
`NewCoordinator` (`internal/agent/coordinator.go:108`), embeds the prompt
description from `ask_user_questions.md`, and adds the tool to the default
allowed-tool list in `config.allToolNames` (`internal/config/config.go:464`).
The accompanying tool description (`ask_user_questions.md`) gives the model
clear guidance on when to ask vs. when to keep going.

## Strengths
- The `ask_user_questions.md` prompt is well-scoped: it explicitly tells the
  model **not** to use the tool to ask for plan feedback (lines 30-32) and to
  put recommended options first with a "(Recommended)" suffix. This is the
  kind of prompt-engineering detail that determines whether agents abuse the
  tool or use it well.
- `mockQuestionsService` (`ask_user_questions_test.go:18`) implements both the
  pubsub broker and the `Ask` method cleanly, and the table-driven test cases
  cover single-select, multi-select, multi-question, and error paths
  (lines 38-128).
- The default-allowed-tool addition (`config/load_test.go:493`) updates the
  golden list explicitly rather than relying on order-insensitive matching;
  this catches accidental reorderings of tool registration.
- `Question`/`Option` types are kept minimal (UUID, label, optional
  description, multi-select bool). Resists the temptation to add
  validation/defaults/templating that would balloon the surface.

## Nits
- `AskUserQuestionParams` (`ask_user_questions.go:19`) has no validation: if
  the model sends `Questions: []` the request will still be created and the
  user will be prompted with an empty dialog. A guard returning a tool error
  ("must include at least one question") would be one line and would catch a
  common LLM failure mode.
- The tool gets added to the *default* allowed-tool list
  (`config/config.go:464`). For users with strict allowlists this is fine, but
  it is also enabled for headless / non-interactive runs where there is no UI
  to surface the prompt. Worth either gating in `NewAskUserQuestionsTool` on a
  TTY/UI-available signal, or noting in the description that calling it in
  headless mode hangs.
- `json.Marshal(resp.Answers)` (`ask_user_questions.go:34`) discards
  `resp.RequestID`. The `RequestID` is asserted in the tests but never
  reaches the model. If the model ever needs to correlate batched
  questions, only the per-`Answer` `ID` field is available — fine, but the
  tests at `ask_user_questions_test.go:262` assert
  `tt.mockResponse.Answers` matches `gotAnswers` exactly, which would silently
  break if the response shape ever extends. A targeted assertion on the
  shape (`ID`, `Selected` only) would be more refactor-safe.
- The new service is wired into pubsub (`app/app.go:483`) but I do not see a
  shutdown path for in-flight `Ask` calls. If the user closes the session
  while the model is awaiting an answer, what happens? A sentence in the PR
  description or a deferred `service.Close()` would close that loop.

## Recommendation
Land after adding the empty-array guard and clarifying behavior in
non-interactive sessions. The core design is right and the test coverage is
solid for a feature this size.
