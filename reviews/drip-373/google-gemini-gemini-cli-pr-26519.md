# google-gemini/gemini-cli #26519 — fix(core): retry on ERR_STREAM_PREMATURE_CLOSE errors

- **Head:** `066f16f` (`066f16f23136e883499b7673da057013fd47d777`)
- **Repo:** google-gemini/gemini-cli
- **Scope:** `packages/core/src/utils/retry.ts:61` (one-line allowlist append) + `packages/core/src/core/issue_25253.test.ts` (new 154-line regression test) + drive-by deletion of `.gemini/settings.json`

## What the diff does

Adds `'ERR_STREAM_PREMATURE_CLOSE'` to the `RETRYABLE_NETWORK_CODES` array at `retry.ts:61`. Pairs it with a new test file at `packages/core/src/core/issue_25253.test.ts` that wires up a `GeminiChat` instance with a mocked `ContentGenerator` whose `generateContentStream` yields one chunk on the first call, throws an `Error` with `code = "ERR_STREAM_PREMATURE_CLOSE"`, then on the second call yields a second chunk. The test asserts that:

- `callCount === 2` (retry actually fired)
- a `StreamEventType.RETRY` event was emitted
- two `StreamEventType.CHUNK` events were emitted (one from each attempt)
- no `error` event was emitted

Also unrelated-but-fine: deletes `.gemini/settings.json` (an experimental-features file that should not be checked in).

## Why this is correct

`ERR_STREAM_PREMATURE_CLOSE` is the Node `node:stream` synthetic error raised by `finished()` / `pipeline()` when the underlying stream emits `close` before `end`. For Gemini SDK streaming responses this maps to "the upstream HTTP/2 connection was reset mid-response". This is precisely the class of transient network failure the existing `RETRYABLE_NETWORK_CODES` list (`ECONNRESET`, `ETIMEDOUT`, `EAI_AGAIN`, `UND_ERR_HEADERS_TIMEOUT`, `UND_ERR_BODY_TIMEOUT`, `UND_ERR_CONNECT_TIMEOUT`) was built for. The omission was an oversight tied to the SDK's switch from raw fetch to `node:stream`-based readers.

## Strengths

- **Test reproduces the exact issue #25253 path** — mid-stream throw, not a top-of-call connection failure. This pins the harder case (the one where some bytes were already delivered to the consumer).
- **The `(prematureCloseError as any).code = '...'` assignment at `:106-107`** mirrors how Node actually decorates these errors (it's a string prop, not a thrown subclass), so the test matches production reality.
- **Allowlist-based fix is the right surface.** `retry.ts` already has the well-defined classification function; piggybacking on the existing list rather than adding a new pattern-match keeps classification centralized.

## Concerns

1. **Mid-stream retry semantics are subtle.** The test at `:139-149` asserts that both chunks make it through to the consumer. That's the desired user-visible behavior, but it means partial output from attempt 1 (`'Part 1'`) AND complete output from attempt 2 (`'Part 2'`) are both emitted. If attempt 2 starts the response over (most LLM streaming APIs do not resume), the consumer sees `Part 1Part 2` where the canonical answer would have been just `Part 2`. This is acceptable — better than the status quo of a hard error — but worth a one-line note in the PR body that callers should be aware completed chunks from a retried-then-succeeded stream are NOT discarded. A `RETRY` event between them is the only signal.
2. **Test file naming.** `issue_25253.test.ts` is descriptive but the codebase pattern (judging from sibling files) favors `<feature>.test.ts`. A `geminiChat.retry.test.ts` filename + `it('retries on ERR_STREAM_PREMATURE_CLOSE during streaming', ...)` would be more discoverable for future regression hunts. Trivial.
3. **Drive-by deletion of `.gemini/settings.json`** is unrelated and should ideally be its own commit — but reasonable, that file should not be in git regardless.
4. **No explicit cap on retry count for this code.** It inherits the existing `getMaxAttempts()` cap (3) so an HTTP/2 endpoint that consistently resets every stream will fail loudly after 3 attempts. Good.

## Verdict

**merge-as-is** — one-line allowlist extension, well-tested, addresses a real issue with a documented repro. The test naming and partial-stream-semantics nits are not worth blocking on. Head `066f16f`.
