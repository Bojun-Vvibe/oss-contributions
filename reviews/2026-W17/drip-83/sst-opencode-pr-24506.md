# sst/opencode PR #24506 — fix: disable tools when MAX_STEPS limit is reached

- Repo: sst/opencode
- PR: #24506
- Head SHA: `b90ec230`
- Author: @alfredocristofano
- Diff: +1/-1 across 1 file (`packages/opencode/src/session/prompt.ts`)
- Closes: #24505

## What changed

One-line fix at `packages/opencode/src/session/prompt.ts:1495`. When `isLastStep` is true the assistant is already prefilled with a `MAX_STEPS` sentinel message telling the model "tools are disabled now," but the underlying `handle.process({ tools, ... })` call was still passing the full `tools` object. The change swaps `tools` for `tools: isLastStep ? {} : tools`, so on the terminal step the model literally cannot emit any tool calls.

## Specific observations

- The fix is consistent with the prefilled assistant message that was added on the same call (`messages: [...modelMsgs, ...(isLastStep ? [{ role: "assistant" as const, content: MAX_STEPS }] : [])]`, lines 1492-1494). Without `tools: {}` the model could still cheerfully ignore the prose hint and emit a tool call anyway — so the prose+tools-empty pair is the right shape.
- `toolChoice` directly below stays `format.type === "json_schema" ? "required" : undefined`. Worth one sanity check: for a `json_schema` final-step run, `toolChoice: "required"` against an empty `tools: {}` map will fault on most providers (OpenAI rejects required-tool-choice with no tools). In practice `isLastStep` is unlikely to coincide with `json_schema` format because the JSON-schema path is the one-shot terminal solver, but a one-line guard `toolChoice: format.type === "json_schema" && !isLastStep ? "required" : undefined` would prevent a future regression where the two paths cross.
- No test added. A 3-line vitest that runs the prompt path with `isLastStep=true` and asserts the captured `tools` arg is `{}` would lock this in cheaply, since the file already mocks `handle.process` elsewhere.

## Verdict

`merge-after-nits`

## Rationale

Correct, minimal fix that makes the prefilled "tools disabled" message actually true. The `toolChoice: "required"` interaction is a small future-proofing nit, not a blocker — the JSON-schema path is unlikely to hit `isLastStep` in current flows.
