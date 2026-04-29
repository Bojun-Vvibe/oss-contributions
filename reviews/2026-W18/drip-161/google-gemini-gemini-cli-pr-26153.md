# google-gemini/gemini-cli PR #26153 — Respect logPrompts flag for logging sensitive fields

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26153
- Head SHA: `25082cd3a0fa098ee0ee1422b2f2c5ee2ac6e4b2`
- State: OPEN, +437/-57 across 4 files

## What it does

Privacy-correctness fix: when `logPrompts: false`, several telemetry events were *still* emitting user-sensitive content to OTEL log records and Clearcut, contradicting the documented privacy contract. PR gates the offending fields behind `getTelemetryLogPromptsEnabled()`, matching the existing pattern that `UserPromptEvent`, `HookCallEvent`, and the `toSemanticLogRecord` paths already follow. All conditionally-included fields also get an existence check, mirroring the `getTelemetryLogPromptsEnabled() && this.xxx` shape used in `toLogRecord`.

Affected event classes:
- `ApiRequestEvent.toLogRecord` — `request_text` (full conversation contents)
- `ConsecaPolicyGenerationEvent` — `user_prompt`, `trusted_content`, `policy`
- `ConsecaVerdictEvent` — same plus `verdict_*` fields
- One more class referenced in the body (likely a Clearcut equivalent of the above)

## Specific reads

- `packages/core/src/telemetry/conseca-logger.test.ts:148-318` — the load-bearing test surface. Four new tests in this file pin the matrix:
  1. `'should omit user_prompt/trusted_content/policy from OTEL when logPrompts is disabled'` — constructs a `configNoPrompts` mock with `getTelemetryLogPromptsEnabled: () => false`, fires `ConsecaPolicyGenerationEvent('sensitive prompt', 'sensitive content', 'sensitive policy')`, asserts `attrs['user_prompt'] / ['trusted_content'] / ['policy']` are *all* `undefined`. This is the right shape — testing for absence, not for empty string or sanitized value.
  2. The Clearcut variant asserts `mockClearcutLogger.createLogEvent` is called with `[{gemini_cli_key: EventMetadataKey.CONSECA_ERROR, value: 'some error'}]` — i.e., only the non-sensitive `error` field survives. This pins the asymmetry that `error` is *not* gated (operator-debugging signal vs user content), which is the right product call.
  3. The "logPrompts is enabled" inverse test pins the positive case (`expect(attrs['user_prompt']).toBe('visible prompt')`), so a future regression that *over*-redacts also trips a test.
  4. The verdict OTEL gate test mirrors the same shape for the verdict event class.
- The four-class matrix (4 events × {OTEL, Clearcut} × {disabled, enabled}) implies up to 16 test variants. Diff says +437 with the test file holding most of it, which is the right ratio for a privacy-class fix — testing the disabled-leg explicitly is the entire point.
- The pattern `getTelemetryLogPromptsEnabled() && this.xxx` (per the PR body) is a defensive-truthiness shape: even when prompts *are* enabled, `undefined`/empty values are still omitted from the attribute bag rather than emitted as the literal string `"undefined"`. That matches the OTEL convention of "absent = unknown, present = value".

## Risk

- Behavioral change for downstream OTEL consumers who were *relying* on the broken behavior to access prompt content even with `logPrompts: false`. That's a privacy-policy-violating reliance and the right call to break, but worth a CHANGELOG entry callout for operators.
- The four event classes touched are all in the `conseca` (consecutive-policy) telemetry path. Worth a one-line audit of *all* `*Event.toLogRecord` and `*Event.toSemanticLogRecord` methods in `packages/core/src/telemetry/` to confirm no other event class has the same bug. The PR body says "consistent with the existing pattern used by `UserPromptEvent`, `HookCallEvent`" — so a grep for `toLogRecord(` returning records that include any of `prompt|content|policy|message` field names without a `LogPromptsEnabled` gate would surface any other miss.
- Test mocks for `Config` are inline duck-typed (`as unknown as Config`) per test. That's pragmatic, but a shared `makeConfigMock({ logPrompts: false })` factory would shrink the +437 to closer to +200 and reduce "did I forget to mock `getTelemetryTracesEnabled`?" drift.

## Verdict

`merge-after-nits` — exactly the right shape of fix for a privacy-class bug: gate at the emit site with a both-legs-tested matrix, preserve operator-debug fields (`error`) explicitly. Two nits: (1) factor the repeated `configNoPrompts` mock construction into a shared `makeConfigMock(overrides)` helper to cut the test-file duplication, (2) sweep for other `*Event.toLogRecord`/`*Event.toSemanticLogRecord` sites with the same shape and either fix in this PR or file a follow-up issue with the grep result. Optional: a CHANGELOG line flagging the behavioral fix for downstream OTEL consumers.
