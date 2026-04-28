# sst/opencode #24754 — fix: SSE stream hang causing run --format json to hang indefinitely

- PR: https://github.com/sst/opencode/pull/24754
- Head SHA: `ae6327153f8c`
- Diff: 31+/1- across 3 files (`packages/opencode/src/provider/provider.ts`, `packages/opencode/src/session/message-v2.ts`, `packages/opencode/test/session/retry.test.ts`)
- Base: `dev`

## Verdict: merge-as-is

## Rationale

- **Closes a real "infinite hang on dead SSE stream" bug end-to-end.** Today, `opencode run --format json` (and any non-TTY caller) that hits an SSE stream where the upstream goes silent — provider crash mid-stream, network black-hole between proxy and provider, or just an idle-but-not-closed connection — will hang indefinitely. The PR walks the three layers needed to convert that hang into a retry: (a) provider.ts:1413 sets a default 60s `chunkTimeout` when none is configured (was `undefined`, which the upstream Bun fetch treats as "no timeout"); (b) message-v2.ts:970-979 maps the resulting `Error("SSE read timed out")` into an `APIError` with `isRetryable: true` and a typed `code: "SSE_TIMEOUT"` metadata cell; (c) the retry classifier picks up `isRetryable: true` and reissues. The three-layer fix is the right shape — fixing only one layer would leave a different hang or silent-drop hole.
- **The 60s default is a defensible choice for SSE.** Most LLM streaming providers send keepalive comments / heartbeat events at 5-30s intervals; a 60s idle threshold is comfortably above that envelope, and short enough to recover within a single retry budget. The `?? 60_000` syntax preserves operator override (an explicit `chunkTimeout: 0` would still disable, an explicit larger value would widen the budget). Good place to put the policy.
- **Error-mapping branch is precise.** The classifier at `message-v2.ts:970-979` matches on **both** `e instanceof Error` AND `e.message === "SSE read timed out"` — that exact string is what Bun fetch's chunkTimeout path raises, and matching the literal message means the classifier won't accidentally swallow other timeout-shaped errors (e.g. socket-level `ETIMEDOUT`). The `metadata.code: "SSE_TIMEOUT"` cell at `:976` gives downstream observability a stable label to filter on, separate from the human-readable `message`.
- **`isRetryable: true` is the right policy choice.** SSE idle-timeout almost always means "the upstream went away mid-stream" — exactly the kind of transient failure that should retry. The alternative (treating it as a hard error and surfacing to the caller) would force every CLI/API consumer to implement its own retry loop, which is wrong because the existing `SessionRetry.retryable` machinery already centralizes the policy.
- **Tests cover both the classifier and the retry side.** `retry.test.ts:187-196` constructs an `APIError` with the new metadata shape and asserts `SessionRetry.retryable(error)` returns the truthy "SSE stream idle timeout" reason. `:264-272` hits the other end — constructs a raw `Error("SSE read timed out")`, runs it through `MessageV2.fromError(..., {providerID: ProviderID.make("openai")})`, and asserts (a) it produces an `APIError`, (b) `isRetryable === true`, (c) the message is the rewritten "SSE stream idle timeout" string, and (d) `metadata.code === "SSE_TIMEOUT"`. That four-tuple is exactly the wire contract — any future drift breaks the test.

## Nits / follow-ups

- **Comment at `provider.ts:1413` would help.** The `?? 60_000` literal is load-bearing — without context, a future contributor may "tighten" it to 10s or "loosen" to 5min on instinct. A one-line comment naming "default SSE idle threshold; raise via `chunkTimeout` if your provider's keepalive interval exceeds 60s" would prevent that thrash.
- **No test for the `chunkTimeout` default itself.** The classifier and retry-mapping cells are tested, but the `?? 60_000` line at `provider.ts:1413` is asserted only by inference. A small unit test that constructs a provider with no `chunkTimeout` and asserts the resolved value is `60_000` would lock that knob.
- **Worth checking `code` vs `message` in `metadata`.** The literal at `:976` says `code: "SSE_TIMEOUT", message: e.message` — the inner `message` is the raw Bun string, the outer `data.message` is the rewritten human label. That layering is correct but worth a `// inner=raw bun string for forensics; outer=user-facing label` comment so the asymmetry is intentional, not accidental drift.
- **No CHANGELOG entry / docs surface.** Operators running `opencode run --format json` in CI today may have built tolerance around the prior hang behavior (e.g. external timeouts wrapping the call). The new behavior — auto-retry instead of hang — is strictly better but worth a release note.

## What I learned

The "hang forever vs retry" choice on SSE idle timeout is a textbook fail-loud-vs-fail-quiet decision, and the PR makes the right one. The structural lesson: the `chunkTimeout` default + the error classifier + the retry-policy classifier are three layers that *all* have to agree on the contract for retry to actually fire. A common bug in this kind of fix is to plumb the timeout but forget the classifier, leaving a "now you get a hard error instead of a hang, which is worse for the CLI consumer". The triple-cell test at `:264-272` is exactly the assertion that prevents that future regression.
