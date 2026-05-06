# Review: anomalyco/opencode#25977 — fix(acp): return stopReason: cancelled (not end_turn) on user cancel

- Head SHA: `f9502791b16dd77e7488c867352834a0579b3e09`
- Files: 2 (+117 / -4)
- Author: truenorth-lj (LJ Li)
- Verdict: **merge-as-is**

## Summary

Fixes the ACP `session/prompt` RPC so that user-initiated `session/cancel` produces `stopReason: "cancelled"` (per `zStopReason` in the SDK schema) instead of being indistinguishable from `"end_turn"`, and omits the misleading all-zero `usage` field on cancel since the AI SDK never emitted `finish-step` to populate token counts. Closes #25899.

## Specific evidence

- **Detector predicate** at `packages/opencode/src/acp/agent.ts:1483-1484`:
  ```ts
  const wasCancelled = (msg: AssistantMessage | undefined): boolean =>
    msg?.error?.name === "MessageAbortedError"
  ```
  Tag is set in `session/processor.ts` halt path via `MessageV2.fromError` mapping `DOMException("AbortError")` → `MessageAbortedError`. The PR body's claim that this tag is wire-stable in `packages/sdk/js/src/v2/gen/types.gen.ts` is correct — that file is generated from the OpenAPI schema, so renaming the error variant would be a coordinated SDK-version bump.
- **Both return sites** updated at `agent.ts:1503-1507` and `:1527-1531`:
  ```ts
  stopReason: cancelled ? ("cancelled" as const) : ("end_turn" as const),
  usage: msg && !cancelled ? buildUsage(msg) : undefined,
  ```
  Symmetric `as const` annotations preserve the literal-type narrowing so callers' exhaustive switches still typecheck.
- **Third return site is correctly skipped** — the PR body explicitly notes the top-level commands (`/compact` etc.) synthesize their own response without a message and are unchanged. Verified: only the two `if (!cmd)` / command branches return a message-derived response; the others return synthesized envelopes.
- **Test coverage** at `test/acp/event-subscription.test.ts:727-822` (+95 lines, two new tests):
  1. `prompt() returns stopReason: cancelled with no usage when message has MessageAbortedError` — fakes `sdk.session.prompt()` returning a tokens-zero message with `error.name === "MessageAbortedError"` and asserts `result.stopReason === "cancelled"` and `result.usage === undefined`.
  2. `prompt() returns stopReason: end_turn with usage on normal completion` — explicit no-regression for the happy path.
- **PR body cites internal staging E2E** with a real `session/cancel` mid-bash run, with pre-fix wire trace observing `stopReason: "end_turn"` + `totalTokens: 0` and post-fix observing `"cancelled"` + omitted `usage`.

## Why merge-as-is

Two lines of behavior change, both grounded in the wire-stable tag set by the existing halt path. The asymmetry in the prior code was a real bug: zero-token usage reports for cancelled turns are not just semantically wrong — they actively mislead spend dashboards because the provider has already charged for prompt tokens that were sent before cancel. Tests cover both new and existing semantics, and the third return site is correctly out of scope per the PR body's explicit reasoning.

The PR author's deferral of the larger fix (reading the AI-SDK `usage` promise during interrupt to report real input-token counts on cancel) is the right call — that is provider-specific and the stop-reason gap stands on its own.
