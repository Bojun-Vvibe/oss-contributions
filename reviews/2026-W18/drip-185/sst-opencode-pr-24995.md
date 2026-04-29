---
pr: sst/opencode#24995
sha: 6965c11cece3d1821861615a664a1f33a9b790b6
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# feat(session): add question tool instructions to gemini.txt system prompt

URL: https://github.com/sst/opencode/pull/24995
Files: `packages/opencode/src/session/prompt/gemini.txt`
Diff: 1+/1-

## Context

The Gemini system prompt at `gemini.txt:30` instructed the model to "ask
concise, targeted clarification questions" but never named the structured
`question` tool. Result: Gemini emitted plain-text questions inline,
bypassing the multi-choice UI that other agents (`trinity.txt`) get for
free. Closes upstream issue #24994 and is the same gap class as #22244.

## What's good

- One-line addition at `gemini.txt:30` appends "using the `question`
  tool" to the existing "ask concise, targeted clarification questions"
  instruction. Surgical: no other prose change, no behavioural drift on
  any path that didn't already trigger clarifying questions.
- The screenshots in the PR body verify behaviour-flip: pre-change the
  model emits `"What styling approach would you like?"` as plain text;
  post-change it invokes the structured tool and the user gets the
  multi-choice picker. That's a correct end-to-end manual test for a
  prompt-only change.
- `trinity.txt` already uses the same "using the `question` tool"
  pattern, so this is conforming to an established prompt convention
  rather than introducing one.
- Author called out the prior closed attempt #8982 and the related
  unaddressed gap #22244 — useful archaeology for the reviewer and the
  next person working in this area.

## Nits

- None blocking. Optional follow-up: a doctrine note in `prompt/README.md`
  or equivalent stating "every system prompt that mentions clarification
  questions MUST name the `question` tool" would prevent this exact gap
  from re-appearing in the next provider prompt added.
- A grep audit of all `prompt/*.txt` files for "clarification question"
  / "ask question" / "ask the user" without a corresponding tool name
  reference would surface any sibling gaps in one pass.

## Verdict reasoning

One-character-effective, prompt-only change that closes a documented
issue with screenshot-verified behaviour flip. Conforms to an existing
in-repo pattern (`trinity.txt`). Zero risk to non-Gemini paths and
zero compile-time/test surface impact. Atomic, reversible if the
question tool is later renamed. The follow-up nits are doctrine-level,
not blocking on this PR.
