# sst/opencode#24762 — fix: SSE stream hang causing `run --format json` to hang indefinitely

- Head SHA: `792366b`
- Author: bingkxu
- Files: 3 / +31 / −1
- Closes #8002

## Summary

Activates the existing `wrapSSE()` idle-timeout protection in `provider.ts` by defaulting `chunkTimeout` to 60s (it was previously `undefined` because no provider sets the option), and threads the resulting `"SSE read timed out"` error through `message-v2.ts` `fromError` → `APIError(isRetryable: true)` so the existing exponential-backoff retry chain (2s→4s→8s→16s→30s cap) kicks in automatically when the satellite/mobile/unstable-link SSE socket silently dies (no RST/FIN, `reader.read()` Promise never resolves).

## Specific observations

- Load-bearing one-line fix at `packages/opencode/src/provider/provider.ts:1449` — `const chunkTimeout = options["chunkTimeout"] ?? 60_000` — every other byte of the wrap path was already correct; the bug was that the default never armed the timer. Right shape: `??` (not `||`) preserves the ability to pass `0` to disable the timeout if a provider wants to opt out, though no test pins that semantics.
- New `case e instanceof Error && e.message === "SSE read timed out"` arm at `packages/opencode/src/session/message-v2.ts:1118-1126` mapping the wrap-emitted error to `APIError({ isRetryable: true, metadata: { code: "SSE_TIMEOUT", ... } })`. The `isRetryable: true` is what makes the existing `SessionRetry.retryable(error)` matcher upstream pick this up — clean, narrow integration with no policy duplication.
- Test surface at `packages/opencode/test/session/retry.test.ts:233-244` and `:332-339` covers both the round-trip `fromError("SSE read timed out") → APIError → SessionRetry.retryable` (ensures the new arm is reachable from the dispatch surface) AND the direct `SessionRetry.retryable` classification on a hand-constructed `APIError` (pins the boundary contract). The dual cell is the right shape — without the round-trip test, a future rename of the substring `"SSE read timed out"` would silently break the integration.
- The 60s default is hard-coded as a magic number — no constant, no comment naming why 60s instead of 30s/120s, no link to the issue's "common on satellite, mobile" rationale. Future tuning will require grepping.

## Verdict

`merge-after-nits`

## Rationale

Right-shape minimal fix activating an already-shipped protection rather than inventing new infrastructure. Surgical 2-line behavior change + 2-cell test is the correct review surface. Nits: (1) extract the `60_000` to a named constant `DEFAULT_SSE_CHUNK_TIMEOUT_MS = 60_000` with an inline comment naming the satellite/mobile rationale and the `?? 0`-disables-it semantics; (2) add a third test cell pinning that explicit `chunkTimeout: 0` still disables the timer (so future refactors don't silently re-break the override); (3) consider matching on a structured error code rather than the brittle exact-string `"SSE read timed out"` substring — any rename in the wrap-emitter side will silently regress the retry path. None block merge.

