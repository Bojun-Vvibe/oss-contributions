# Review: QwenLM/qwen-code #3807 — fix(telemetry): suppress async resource attribute warning on startup

- **Repo**: QwenLM/qwen-code
- **PR**: #3807
- **Head SHA**: `4fb481b9762ae26ece2e2cd77f3916ebb68a4a8f`
- **Author**: doudouOUC

## What it does

Disables OpenTelemetry's async resource detectors
(`autoDetectResources: false`) on the `NodeSDK` instance. Without this,
`HttpInstrumentation` span creation can read resource attributes before
the host/process/env detectors settle, triggering an
`OTel diag.error`.

## Diff notes

- `packages/core/src/telemetry/sdk.ts:248-251` — adds
  `autoDetectResources: false` to the `NodeSDK({...})` constructor with
  a precise inline comment explaining the failure mode (pending
  attributes + diag.error during instrumented span creation). The
  comment is tight and accurate — exactly the "why" you want to find
  six months later.
- `packages/core/src/telemetry/sdk.test.ts:139,162,301` — three test
  cases extended with
  `expect(NodeSDK).toHaveBeenCalledWith(expect.objectContaining({ autoDetectResources: false }))`.
  Covers the gRPC, HTTP-with-signal-paths, and "no-exporters" branches.
  Solid: any future refactor that drops the flag fails three tests.

## Concerns

1. **Loss of detected attributes**: The host/process/env attributes
   (hostname, process pid, runtime version, OS) are useful for telemetry
   triage. Disabling all auto-detection is a sledgehammer. OTel exposes
   per-detector toggles via `resourceDetectors: [...]` — passing in only
   the *sync* detectors (e.g., `envDetectorSync`, `processDetectorSync`)
   would suppress the warning while keeping the metadata. Worth asking
   the author whether they considered the granular path before merging
   this.
2. **No reproducer in the PR description**: A 2-line "before/after diag
   output" snippet would make this much easier to review. As-is, we're
   trusting the inline comment.
3. **Resource-attribute consumers downstream**: If any dashboard or
   alerting depends on `host.name` / `process.pid` being on telemetry
   spans, this PR silently breaks them. A maintainer with telemetry
   ownership should sign off.

Mechanically the change is correct and well-tested. The trade-off is
the policy question.

## Verdict

needs-discussion
