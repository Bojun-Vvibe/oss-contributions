# charmbracelet/crush #2773 — fix(ui): include cancelled prompts in arrow-up history

- **Repo:** charmbracelet/crush
- **PR:** https://github.com/charmbracelet/crush/pull/2773
- **HEAD SHA:** `bafe8f8c414d4a130e770e70a178718cdbf0ec32`
- **Author:** pragneshbagary
- **Verdict:** `merge-after-nits`

## What the diff does

Single-file fix at `internal/ui/model/ui.go` closing the symptom
"prompt I sent and then ESC-cancelled before completion is missing
from arrow-up recall":

1. New typed bubbletea message at `ui.go:160-163` —
   `agentRunCompleteMsg struct{}` with a comment locking the dual
   purpose ("normally or via cancellation").
2. New Update arm at `:593-594` — `case agentRunCompleteMsg: cmds = append(cmds, m.loadPromptHistory())`
   wires the message to a history reload.
3. `sendMessage` at `:3156-3173` — both the `context.Canceled`
   branch and the success branch now `return agentRunCompleteMsg{}`
   instead of `return nil`, so the history-refresh fires on either
   completion path.

## Why the change is right

Diagnosis is correct: prompt history is persisted at *send*-time
(otherwise this fix wouldn't recover it), but the in-memory
arrow-up buffer was only refreshed on the success path. Cancellation
returned `nil` from the bubbletea command, no `agentRunCompleteMsg`
fired, no `loadPromptHistory()` call, and the cancelled prompt sat
in the persisted store invisible to the next `↑` keystroke until the
next normal completion happened to refresh.

Putting the refresh at a typed message rather than wiring
`loadPromptHistory()` directly into both code paths is the right
Elm-style shape — `agentRunCompleteMsg` is a single observable end-of-
turn signal that future paths (timeout, error-then-recover, etc.)
can also emit if the same refresh semantics apply, without each
caller having to remember to fan out the side effect.

The `nil → agentRunCompleteMsg{}` change at `:3166` (cancel branch)
is the load-bearing line; the success-path swap at `:3173` is
behaviorally identical (both end in a refresh, just now via the
typed message rather than whatever previously triggered it) and is
the cleanup that makes the new path symmetrical with the old.

## Nits (non-blocking)

1. **Error path still returns `util.InfoMsg`** at `:3168-3171` rather
   than ALSO emitting `agentRunCompleteMsg{}`. If the agent run
   fails after the prompt was persisted (most failure modes), the
   prompt is in the store but arrow-up still won't see it until the
   next refresh. Consider `return tea.Batch(util.InfoMsg{...},
   agentRunCompleteMsg{})` or splitting into two separate Cmds —
   the symptom this PR fixes recurs identically on the error branch.

2. **No regression test.** `internal/ui/model` likely has bubbletea
   model tests that drive Update with a sequence of messages and
   assert on the model's `promptHistory.entries` length. A test
   asserting "send + cancel + arrow-up returns the cancelled prompt"
   would pin the symptom and prevent a future code path from
   `return nil`-ing again on cancel.

3. **`loadPromptHistory()` reload semantics.** If `loadPromptHistory`
   re-reads from disk every time, every turn completion now triggers
   a disk read. For long sessions (100+ prompts) this could add
   measurable per-turn latency on cold filesystems. If the loader
   is in-memory or already cached this is a non-issue — worth a
   one-line confirmation in the PR description.

4. **Comment polish at `:160-162`** — "agentRunCompleteMsg is sent
   when an agent run finishes (normally or via cancellation)" is
   good; consider adding "**not** on agent error" so the asymmetry
   in nit #1 is at least documented if the author chooses not to
   address it now.

## Verdict rationale

Right diagnosis, right shape (typed end-of-turn message + single
Update arm), load-bearing change is the cancel-path swap. The
asymmetric error-path handling is the only real review concern;
test coverage and comment polish are minor.

`merge-after-nits`
