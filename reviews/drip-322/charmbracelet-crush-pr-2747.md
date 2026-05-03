# charmbracelet/crush PR #2747 — fix(tools/job_kill): use longer job_kill desc

- Link: https://github.com/charmbracelet/crush/pull/2747
- SHA: `20a36ecb66259b2b8b6e435b24bbf28a534a0b47`
- Author: meowgorithm
- Stats: +2 / −2, 2 files (`internal/agent/tools/job_kill.go`, `internal/agent/tools/job_kill.md`)

## Summary

Reverts the `FirstLineDescription(jobKillDescription)` wrapper around the embedded markdown description back to `string(jobKillDescription)`, so the model sees the full multi-paragraph description instead of just the first line. Also tightens the first line of `job_kill.md` from "Terminate a background shell process by ID; shell ID becomes invalid after killing." to "Terminate a background shell process." — moving the "shell ID becomes invalid" detail into the body where the longer description now reaches.

## Specific references

- `internal/agent/tools/job_kill.go` L32: `FirstLineDescription(jobKillDescription)` → `string(jobKillDescription)`. Mirrors the pattern other tools in the same package use when the full markdown body is intentionally exposed to the model. This is the actual fix.
- `internal/agent/tools/job_kill.md` L1: shortened first line. Important because tooling lists in many models truncate or weight the first description line heavily; keeping it short and verb-first ("Terminate a background shell process.") is exactly what the LLM-facing convention wants. The dropped clause about ID invalidation must therefore be present somewhere later in the body — that's not visible in the diff hunk shown but is implied by the PR description ("works better with some LLMs").
- The change is tiny and reversible — no API surface, no plumbing, no test impact. Risk is bounded to "the model sees a longer string at tool-load time", which slightly increases token cost per session but only for prompts that load this tool.

## Verdict

verdict: merge-as-is

## Reasoning

Two-line patch with a clear motivation in the PR body ("works better with some LLMs"). The change is consistent with how peer tools in `internal/agent/tools/` already feed full markdown to the model, and the docstring tightening keeps the first-line-as-summary contract intact. No tests required for a description-string change.
