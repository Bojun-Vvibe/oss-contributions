# charmbracelet/crush #2555 — fix: use small model for task agent instead of large

- **Head SHA:** `147265dae12b6ce624ba2e96518dda999d4d596b`
- **Base:** `main`
- **Author:** talha7k (Talha Khan)
- **Size:** +1 / −1 across 1 file (`internal/config/config.go`)
- **Verdict:** `needs-discussion`

## Summary

One-character config change at `internal/config/config.go:530`:
```
-			Model:        SelectedModelTypeLarge,
+			Model:        SelectedModelTypeSmall,
```
Changes the default model selection for the built-in `Task` agent
from "large" to "small". The `Task` agent is described in this same
struct as "An agent that helps with searching for context and finding
implementation details" with read-only tools and no MCPs/LSPs by
default.

## What's right

- **The agent description matches the small-model use case.**
  Searching for context and finding implementation details is mostly
  classification + grep + retrieval — not deep reasoning. The
  `AllowedTools: resolveReadOnlyTools(allowedTools)` line directly
  above (`config.go:533`) tells you the agent can't write or execute,
  just read. Read-only context-gathering is the canonical "small
  model is fine" workload.

- **Cost and latency win is real.** A `Task` agent invoked
  recursively (which is the common pattern for spawning subagents to
  explore parts of a codebase) on the large model can dominate
  per-session token spend. Routing it to small saves real money and
  shaves seconds off each subagent call. Same reasoning as qwen-code
  #3848 and gemini-cli's steering-ack pattern: background
  classification → fast/small model.

- **One-line change, easy to revert.** If quality regresses, the
  rollback is trivially `git revert`.

## Concerns / why this needs discussion

- **No PR description, no benchmark, no eval data.** The PR is a
  one-line change with no body, no rationale, no quality comparison
  between large-model task subagents and small-model task subagents.
  For a change that affects every Crush user's default agent
  behavior, that's not enough. Even a paragraph saying "I ran N
  subagent searches against repos X, Y, Z and the small model got
  recall above T%" would be enough to ground the discussion.

- **"Task agent quality" is exactly the dimension where small models
  can fail silently.** The Task agent's job is to *find the right
  files / functions* to feed back to the parent agent. If a small
  model misses a relevant file (false negative), the parent agent
  proceeds without that context and produces a subtly worse answer.
  This failure mode is hard to spot in casual testing but shows up
  in eval harnesses. Without an eval delta, "looks fine on my
  machine" is not enough signal.

- **Maintainer policy choice, not a bug fix.** The current
  `SelectedModelTypeLarge` was presumably chosen deliberately when
  the Task agent was added. Flipping it without a changelog entry
  treats a deliberate quality/cost tradeoff as if it were a typo.
  Crush maintainers (e.g., @meowgorithm based on commit history)
  should be the ones to decide where on the quality-vs-cost curve
  the *default* sits — third-party users can already override by
  configuring their own agent.

- **Alternative: make it configurable per-user without changing the
  default.** If the goal is "let users opt the Task agent into the
  small model", that's a different (and more conservative) PR:
  expose the `Model:` field in user config and document the
  tradeoff. Doesn't ship a behavior regression to users who didn't
  ask for it.

- **No tests touched.** Whatever test exercises agent setup in the
  config package presumably still passes (the type didn't change),
  but a small assertion that the Task agent uses the
  *intended* model would be cheap insurance against future flips.

## Risk

- **Quality regression risk: medium.** Affects default behavior of a
  built-in agent that runs recursively in many sessions. Users who
  notice will blame "Crush got dumber"; they may not connect it to a
  one-line config change.
- **Reversibility: trivial.** `git revert` is a one-line change.

## Verdict

`needs-discussion`. The change *might* be the right call, but a
one-line diff with no rationale isn't enough to land a default
behavior change for every Crush user. I'd want either:
1. An eval comparison (recall@k for Task subagent file selection,
   small vs large) showing parity or near-parity, **or**
2. Repositioning the PR as "make Task agent's model configurable" —
   keep large as the default, let users opt down.

If the Crush maintainers say "yeah, we already know small is fine
for this, just merge" — fine, that's their call. But the public PR
shouldn't land without that signal recorded somewhere.
