# Review: sst/opencode #25198 — fix: fix AI refusing to commit

- **PR**: https://github.com/sst/opencode/pull/25198
- **Author**: scarf005 (scarf)
- **Head SHA**: `dbf6fc674349b1c7e70e8d4862dd1a55631bf188`
- **Base**: `dev`
- **Files**: `session/prompt/default.txt` (-2), `session/prompt/trinity.txt` (-2), `tool/bash.txt` (+1/-2)
- **Verdict**: **needs-discussion**

## Reasoning

Closes issue #17157 by removing/softening three lines of system-prompt
policy that the author argues cause the agent to refuse `git commit`
even when `AGENTS.md` explicitly tells it to commit.

Specifically:

1. `session/prompt/default.txt` (around the original line 86) deletes the
   stand-alone sentence stating that the agent must never create commits
   unless explicitly asked.
2. `session/prompt/trinity.txt` (around the original line 78) deletes the
   identical sentence from the Trinity preset.
3. `tool/bash.txt` (around the original line 53) softens the opening
   directive of the "Committing changes with git" block from
   "Only create commits when requested by the user. If unclear, ask first.
   When the user asks…" down to just "When creating a new git commit,
   follow these steps carefully:". A second copy of the never-commit-unless-asked
   bullet is also removed from the Git Safety Protocol bullet list.

Why this needs discussion rather than a straight verdict:

- **The change is a real policy shift, not a bug fix.** The deleted
  sentences are not contradictory or stale — they encode an explicit
  product-safety stance that "agents should not silently commit." Removing
  them is reasonable iff the maintainers want to delegate commit policy
  entirely to the user's `AGENTS.md`, but that is a product call, not a
  prompt-engineering nit. The PR title frames this as a bugfix; it isn't.
- **The fix doesn't actually require deletion.** The reported failure mode
  (agent refuses to commit even when `AGENTS.md` says to) is a
  precedence-resolution problem between the system prompt and the
  user-supplied `AGENTS.md`. A targeted edit — e.g. adding "unless your
  AGENTS.md instructs otherwise" to the existing sentence — would
  address the user complaint without losing the safety floor.
- **`tool/bash.txt` change has a non-obvious side-effect.** Rewriting the
  opening from "Only create commits when requested by the user. If
  unclear, ask first." to "When creating a new git commit, …" removes
  the `ask first` fallback for the ambiguous case. Even agents that
  respect the deleted bullet will now skip the clarifying-question step,
  which is a real behavior change distinct from the intended fix.
- The author's verification ("simple prompt changes") is thin for a
  change to global system prompts that ship to every session.

## Suggested follow-ups

- Counter-proposal: keep the lines but add an explicit precedence rule:
  "These commit-policy lines apply unless the project's `AGENTS.md`
  explicitly authorizes the agent to commit autonomously." Solves the
  reported issue without removing the safety floor for users who never
  set up `AGENTS.md`.
- If the maintainers do want this deletion, restore the "If unclear,
  ask first." bullet in `tool/bash.txt` — that one is genuinely
  orthogonal to the auto-commit policy and is load-bearing for
  ambiguous cases.
- Add a regression eval against the issue #17157 reproducer to lock
  in whichever direction the maintainers choose.
