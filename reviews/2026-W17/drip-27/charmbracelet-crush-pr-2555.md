# charmbracelet/crush PR #2555 — Use small model for task agent instead of large

- **Repo:** charmbracelet/crush
- **PR:** [#2555](https://github.com/charmbracelet/crush/pull/2555)
- **Head SHA:** `147265dae12b6ce624ba2e96518dda999d4d596b`
- **Author:** talha7k
- **Size:** +1 / −1 across 1 file (`internal/config/config.go`)
- **Reviewer:** Bojun (drip-27)

## Summary

One-character fix in `Config.SetupAgents()`: change the `AgentTask`
sub-agent's `Model` field from `SelectedModelTypeLarge` to
`SelectedModelTypeSmall`. Task agent is the read-only search/retrieval
sub-agent (tools: glob, grep, ls, sourcegraph, view) — heavy reasoning
isn't useful, so spending the large-model price/latency on every
sub-agent invocation was waste. PR closes #427 and is the third
attempt at this fix (after #914 which never merged).

## Key change

### `internal/config/config.go:527–531`

```go
{
    ID:           AgentTask,
    Name:         "Task",
    Description:  "An agent that helps with searching for context and finding implementation details.",
-   Model:        SelectedModelTypeLarge,
+   Model:        SelectedModelTypeSmall,
    ContextPaths: c.Options.ContextPaths,
    AllowedTools: resolveReadOnlyTools(allowedTools),
    // NO MCPs or LSPs by default
}
```

`AgentCoder` (the main coding agent, defined a few lines above) is
untouched and still uses `SelectedModelTypeLarge`. Only `AgentTask`
flips. The PR body explicitly notes this asymmetry as intentional.

## What's good

- Right fix for the right reason. The task agent's tool surface
  (`resolveReadOnlyTools(allowedTools)`) is read-only retrieval —
  glob/grep/ls/sourcegraph/view. None of these benefit from the
  large model's reasoning depth; they benefit from latency and
  throughput, which is what the small slot is for.
- One-line diff with no behavioural change to `AgentCoder` makes
  this trivially auditable. Reviewers can verify correctness by
  reading the surrounding 10 lines and confirming `AgentCoder`
  isn't accidentally collateral.
- The PR also makes the user's `small` model configuration
  *actually take effect* for sub-agents. Per the PR body: "users
  who configured a `small` model in `crush.json` had it completely
  ignored for sub-agents". Fixing this restores the configuration
  surface to what the docs/UX implies.
- Closes a long-standing issue (#427) and supersedes a prior
  attempt (#914). The third-time-lucky pattern usually means the
  earlier attempts had additional scope creep or test breakage —
  worth checking #914 to confirm this version doesn't reintroduce
  whatever blocked the prior PR (see concern 2 below).

## Concerns

1. No new test asserting `AgentTask.Model == SelectedModelTypeSmall`.
   The PR body's "Test plan" claims existing tests in
   `load_test.go` cover this, but I'd expect a one-line table-test
   addition explicitly pinning the choice — otherwise a future
   refactor of `SetupAgents` could silently flip it back to
   `Large` and the existing tests wouldn't catch it. Even
   `assert.Equal(t, SelectedModelTypeSmall, cfg.Agents[AgentTask].Model)`
   in the existing test would suffice.

2. The two prior attempts at this fix (#914 mentioned in the PR
   body, plus the implication that #427 has been open long enough
   to have multiple stalls) suggest there *might* be a reason this
   wasn't landed before — e.g. the small model historically
   couldn't handle the task agent's prompt, or there's a downstream
   call site that assumes the task agent has large-model context
   capacity. A reviewer should verify that the small model's
   typical context window is large enough to hold the task agent's
   typical prompt + retrieved context. If not, the user-visible
   regression would be "task agent now silently truncates results
   on long codebases".

3. Behavioural change for any user who *deliberately* set their
   small slot to a model that's *worse* than their large slot
   (e.g. a tiny local model for cheap completion vs. a hosted big
   model for everything else). For those users, this PR makes the
   task agent get worse — which is technically what they
   configured, so it's the correct behaviour, but it's a silent
   change in behaviour for existing installs and worth a CHANGELOG
   note flagging "the `small` model now drives the task sub-agent;
   if your `small` model is significantly weaker than your `large`
   model, sub-agent results may differ from prior versions".

4. `SelectedModelTypeSmall` and `SelectedModelTypeLarge` are
   stringly-typed enum constants in this package — easy to swap
   the wrong way during a future refactor. Consider whether a
   typed enum (or at least a named function like
   `defaultTaskModel()` returning the value) would make the
   intent more refactor-resistant. Out of scope for a one-line
   fix, but worth mentioning.

## Risk

Low. Single config-default flip, zero call-site changes, opt-out
trivially via user config (`crush.json` `agents.task.model = "large"`
if the codebase exposes that — worth confirming in docs).

## Verdict

**merge-after-nits**

The fix is right and the rationale (cost/latency for read-only
search) is correct. Concern 1 (a one-line assertion in
`load_test.go`) and concern 3 (CHANGELOG note flagging the
behavioural change) are the two easy adds I'd want before merge.
Concern 2 is a verification step the maintainers can do quickly
by looking at what blocked #914 — if that PR died for unrelated
reasons (mergeability, scope), this can land. If it died because
the small model was inadequate at the time, the answer may be
"yes, but small models in 2026 are different from small models in
the prior PR's era" and merge anyway with a note.

## What I learned

The pattern of "main agent uses big model; read-only sub-agent
uses small model" is a generally good default for any
agent-of-agents architecture. The main agent owns reasoning,
planning, and tool orchestration (where bigger models pay off);
sub-agents own retrieval and pre-digestion (where latency and
parallelism matter more than reasoning depth). Worth borrowing
the shape for any code that has a "main" vs. "helper" agent
split — defaulting helpers to the small slot is the right
ergonomic, and exposing per-agent model overrides is the right
escape hatch for users who want different defaults.
