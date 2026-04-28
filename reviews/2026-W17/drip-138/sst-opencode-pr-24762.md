# sst/opencode #24762 — fix: SSE stream hang causing run --format json to hang indefinitely

- PR: https://github.com/sst/opencode/pull/24762
- Author: bingkxu
- Head SHA: `792366bd48b3`
- Base: `dev` · Closes #8002
- Diff: 31+/1- across 3 files (`packages/opencode/src/provider/provider.ts`, `packages/opencode/src/session/message-v2.ts`, `packages/opencode/test/session/retry.test.ts`)

## Verdict: merge-as-is

## Rationale

- **Right diagnosis of a real "process hangs forever on satellite/mobile networks" symptom.** `wrapSSE()` already had idle-timer protection wired in but was a *dead path* because nobody ever set `chunkTimeout` in provider options — `provider.ts:1449` read `options["chunkTimeout"]` and got `undefined`, so the abort timer was never armed. The one-line change at `:1449` (`options["chunkTimeout"] ?? 60_000`) flips the default from "off" to a 60s idle ceiling, which is the canonical "long enough that no provider's longest-thinking response trips it; short enough that real network drops don't sit forever" tradeoff. This is the minimum-cost fix for a structural bug — the protection mechanism existed; it was just feature-flagged off by accident.
- **The classifier addition is the load-bearing second half.** Without `message-v2.ts:1118-1126` (the new `case e instanceof Error && e.message === "SSE read timed out"` arm in `fromError`), the timeout would surface as a generic non-retryable error and the retry pipeline at `SessionRetry.retryable` would never re-invoke. The new arm wraps it in `APIError({ message: "SSE stream idle timeout", isRetryable: true, metadata: { code: "SSE_TIMEOUT", message: e.message } })` — the `code: "SSE_TIMEOUT"` discriminator in `metadata` is the right move because downstream consumers (telemetry, logging, dashboards) need to count "real provider errors" and "we cut a stream" separately.
- **Two-cell test correctly anchors both halves of the contract.** `retry.test.ts:233-244` pins the classifier (constructed `APIError` with `metadata.code: "SSE_TIMEOUT"` → `SessionRetry.retryable` returns the human message), and `:331-340` pins the mapping (raw `Error("SSE read timed out")` → `MessageV2.fromError` → instance-of `APIError` with `isRetryable: true` and `metadata.code === "SSE_TIMEOUT"`). The second test is the load-bearing one because it pins the literal-string match — if a future refactor changed `wrapSSE`'s thrown message to "SSE chunk timeout" the classifier would silently fall through to the catch-all, but the test would catch it on next run.
- **String-match coupling between thrower and classifier is acknowledged risk, not a bug.** The diff doesn't introduce a typed error class for the timeout — but `wrapSSE` is internal to `provider.ts` and its thrown `Error("SSE read timed out")` literal is matched verbatim at `:1118`. Tight coupling, but contained to one package and pinned by the new test.

## Nits / follow-ups

- `60_000` is a magic number at `provider.ts:1449`. Worth promoting to a named constant `DEFAULT_SSE_IDLE_TIMEOUT_MS` next to the other provider defaults for grep-ability — and so a future "configurable per provider" feature has a single override point.
- The `fromError` arm uses `e instanceof Error && e.message === "SSE read timed out"` — symmetric with the `(e as SystemError)?.code === "ECONNRESET"` style in the next arm but worth a one-line comment noting the literal is matched against `wrapSSE`'s thrown message for grep-back when `wrapSSE` is refactored.
- The PR body claims "verified retry mechanism by temporarily setting chunkTimeout to 2s" — that integration anchor isn't in the test file. A higher-level test that drives a fake SSE source through `wrapSSE` with a stubbed timer and asserts the actual `APIError` surfaces (rather than constructing it manually) would catch a future refactor that breaks the wiring without breaking the unit-level fromError contract.

## What I learned

The "feature-flagged-off protection" anti-pattern shows up wherever a defensive mechanism is gated on a config field that nobody sets — the code is *present and tested* but the production effect is `undefined`. The fix shape — flip the default with `??` at the read site — is preferable to "set it in every options object" because it's one line, one diff, and the override path stays open for callers that *do* want a custom timeout. Pairing the default flip with a dedicated metadata code (`SSE_TIMEOUT`) instead of folding into a generic retryable bucket is what makes the change observable end-to-end: dashboards get a new dimension that lets ops distinguish "we're getting hammered by network drops" from "the provider is having a bad day."
