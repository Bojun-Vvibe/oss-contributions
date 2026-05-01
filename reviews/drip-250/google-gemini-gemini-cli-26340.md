# google-gemini/gemini-cli #26340 — fix(core): remove "System: Please continue." injection on InvalidStream events

- URL: https://github.com/google-gemini/gemini-cli/pull/26340
- Head SHA: `c355a4bb70659cc98bd8b2b599db564e17df4dc9`
- Files: `packages/core/src/core/{client,client.test,geminiChat,geminiChat.test}.ts` + 10 callers/configs (+53/-298)

## Context / problem

The `InvalidStream`-on-empty-response auto-recover path has a tangled history:
- **#20897** added an `isGemini2Model` gate on the post-stream injection because Gemini 3 emits empty wrap-up turns intentionally and was being spuriously re-prompted.
- **#24302** removed that gate to enable cross-model self-heal for malformed function calls — *but conflated two distinct mechanisms*: the in-chat content-error retry inside `geminiChat.ts` (the temperature-bumped retry that's actually useful for `MALFORMED_FUNCTION_CALL`) and the post-stream `"System: Please continue."` synthetic-user-message injection inside `client.ts` (the bad one).

The post-stream injection cannot distinguish "model fell off the rails" from "model deliberately produced nothing because the task ended" — the `NO_RESPONSE_TEXT` heuristic that triggers it fires on the latter case too. Result: every clean turn-end with no terminal text gets a phantom `"System: Please continue."` round-trip, plus a redundant model utterance, charged to the user.

## Design analysis

The PR correctly *separates* the two mechanisms and removes only the bad one:

### Removed: post-stream prompt-injection (`client.ts:807-832` deletion, `:601` removed `import logNextSpeakerCheck`)

The `if (isInvalidStream) { if (this.config.getContinueOnFailedApiCall()) { ... } }` block is gone. The recursive `this.sendMessageStream(nextRequest, signal, prompt_id, boundedTurns - 1, true, displayContent)` with `nextRequest = [{ text: 'System: Please continue.' }]` and the `isInvalidStreamRetry` parameter threading it carried, both gone. `InvalidStream` events now propagate to UI/CLI consumers and the turn ends cleanly. The new dispositive test at `client.test.ts:470-488` (`should propagate InvalidStream events without injecting "Please continue." or recursing`) asserts (a) a single `turn.run` call (no recursion), (b) the `InvalidStream` event reaches the consumer.

### Preserved with one carve-out: in-chat content-error retry (`geminiChat.ts:424-430`)

```ts
const isContentError = error instanceof InvalidStreamError;
const isRetryableContentError =
    isContentError && error.type !== 'NO_RESPONSE_TEXT';
const errorType = isContentError
    ? error.type
    : getRetryErrorType(error);

if (isRetryableContentError || (isRetryable && !signal.aborted)) {
    // 4-attempt retry for MALFORMED_FUNCTION_CALL etc.
}
```

This keeps the temperature-bumped retry alive for `MALFORMED_FUNCTION_CALL` (where the model emitted *something* that just didn't parse — a re-roll often recovers), but explicitly excludes `NO_RESPONSE_TEXT`. The reasoning: an empty stream is *not* a "model emitted bad output that might re-roll cleanly"; it's "model intentionally produced nothing." Retrying it is throwing more tokens at the same nondeterministic process hoping for different output. Correct call.

The `geminiChat.test.ts:777-793` test pins the exact behavior: stream that returns no non-thought text (`NO_RESPONSE_TEXT`) is *not* a second `generateContentStream` call, and *no* content retry telemetry is emitted.

### Cleanup that follows from the removal

- `isInvalidStreamRetry: boolean` parameter threaded through `processTurn` / `sendMessageStream` / `_recoverFromLoop` is gone (`client.ts` `:619,627,653,663,674,832,847,851,902` deletions).
- `if (!isInvalidStreamRetry) { this.config.resetTurn(); }` becomes unconditional `this.config.resetTurn()` at `:851-853` — the conditional-skip only existed to avoid double-resetting on the recursive injection path that no longer exists. Right cleanup.
- `continueOnFailedApiCall` config option dropped (`config.ts:341,349,357,366`, `config.test.ts:306-326`). PR confirms no settings-schema, CLI flag, or docs reference it — only `client.ts` ever called `getContinueOnFailedApiCall()`. Net `-7` on `config.ts`. Right.
- All positional callers updated (`nonInteractiveCli.ts`, `useGeminiStream.ts`, `legacy-agent-session.ts`) to drop the now-removed argument; their tests updated to match.

## Risks

- **Behavior change for operators relying on `continueOnFailedApiCall: true`** — PR confirms it was never user-facing (no schema, no docs). But: it was a public field on the `Config` constructor params, which means any *embedder* of `@google/gemini-cli-core` that passes a config object with `continueOnFailedApiCall` will now get a TypeScript error (the field is removed from the interface, not just deprecated). Worth a `@deprecated` shim that accepts but ignores the field for one minor version, with a console warning, before hard-removing. Marked as "noted breaking changes" in the checklist, which is honest, but in core SDK code one minor version of grace is the courteous default.
- **Telemetry loss** — `logContentRetryFailure(this.config, new ContentRetryFailureEvent(4, 'FAILED_AFTER_PROMPT_INJECTION', modelToUse))` at the prior `client.ts:809-815` was the only emit site for the `'FAILED_AFTER_PROMPT_INJECTION'` reason code. After this PR, that reason code is dead. If any dashboards/alerts key off it, they'll see metric absence (silent), not a hard break. Worth a quick confirmation no such dashboard exists, or a one-line PR note: *"the `FAILED_AFTER_PROMPT_INJECTION` content-retry reason code is no longer emitted; downstream dashboards keying off this code should be updated."*
- **Asymmetric retry policy now embedded in code, not documented** — `MALFORMED_FUNCTION_CALL` retries (4 attempts), `NO_RESPONSE_TEXT` does not. Future contributors will see `error.type !== 'NO_RESPONSE_TEXT'` and wonder why this one type is special. Worth an inline comment at `geminiChat.ts:425`: *"NO_RESPONSE_TEXT means the model ended cleanly with no terminal text — retrying just throws more tokens at the same nondeterministic process. Other content errors (MALFORMED_FUNCTION_CALL etc.) are 'model emitted broken output' and benefit from a temperature-bumped re-roll."*
- **`isContentError` evaluated twice** — `isContentError = error instanceof InvalidStreamError` is now used only to gate `isRetryableContentError` and `errorType`; if the result of the gate is the only consumer, the local `isContentError` is dead after the gate evaluation. Tiny nit.

## Suggestions

- Inline comment at `geminiChat.ts:425` explaining the asymmetric retry policy (above).
- One-version `@deprecated` shim on `continueOnFailedApiCall` for embedders before hard-removing the property.
- PR body callout that `'FAILED_AFTER_PROMPT_INJECTION'` content-retry reason code is no longer emitted.
- Sibling test in `client.test.ts` for `MALFORMED_FUNCTION_CALL` confirming the retry is *still* attempted (not just `NO_RESPONSE_TEXT`'s no-retry locked in) — proves the carve-out is asymmetric on purpose.

## Verdict

`merge-after-nits` — correct surgical separation of two conflated mechanisms (post-stream prompt injection vs. in-chat content-error retry), removing the bad one (which couldn't distinguish "model deliberately produced nothing" from "model fell off the rails" and so charged a phantom round-trip on every clean empty turn) while preserving the good one with a `NO_RESPONSE_TEXT`-excluded carve-out. Net `-245` is honest dead-code excision (parameter threading, never-user-facing config option, recursive-call site, retry-telemetry call). Wants `@deprecated` shim for the public-config-field break, an inline comment locking the asymmetric retry rationale, and a PR-body note about the orphaned `'FAILED_AFTER_PROMPT_INJECTION'` telemetry code.
