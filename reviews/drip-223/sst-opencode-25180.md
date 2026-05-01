# sst/opencode#25180 — fix: enable auto-compaction for sub-agents and improve context overflow detection

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/25180
- Head SHA: `35883effb47b`
- Size: +26 / -0 (two files)
- Verdict: **merge-after-nits**

## What the diff actually does

Two files, two complementary fixes for "sub-agent hangs forever on context
overflow":

1. **`packages/opencode/src/provider/error.ts:25–30`** — extends the
   `OVERFLOW_PATTERNS` regex array with three new arms so post-stream
   error-text classification catches more provider variants:
   - `/token limit exceeded/i` — z.ai GLM models
   - `/max.*tokens.*exceed/i` — generic
   - `/input.*too.*large.*model/i` — generic
   These join the existing six patterns (Ollama, Mistral, z.ai
   `model_context_window_exceeded`, etc.) and feed `isOpenAiErrorRetryable`
   so the retry/compact decision tree fires on a wider input set.

2. **`packages/opencode/src/session/processor.ts:543–568`** — adds a new
   *proactive* pre-stream estimator inside the per-turn `Effect.gen` block.
   Before the request goes out:
   - Reads `input.model.limit.context`. If `> 0` (model declares a limit),
     concatenates `streamInput.system` + `streamInput.messages` into a JSON
     payload, calls `Token.estimate(...)` (newly imported at line 13), and
     compares against `Math.floor(contextLimit * 0.85)` as the compaction
     threshold.
   - If estimated tokens ≥ threshold: sets `ctx.needsCompaction = true`,
     logs `"proactive compaction triggered"` with the three numbers, and
     short-circuit-returns `"compact" as const` from the inner `Effect.gen`.

The load-bearing motivation in the PR body is right and matches the diff:
sub-agents (and z.ai-style providers that silently truncate without
error) never reach the post-stream reactive overflow check, so without a
pre-stream estimator they hang. The 0.85 threshold is the same magic
number opencode already uses elsewhere for compaction triggers, so
operational expectation is preserved.

## Why merge-after-nits

The fix is correct, tightly scoped, and addresses a real hang class. The
nits are all about defensibility-of-the-estimate, not the architecture.

## Nits

1. **`JSON.stringify` of the entire system+messages array is the
   estimator's entire input** (`processor.ts:553–556`). For a long
   conversation this is O(N) on every turn, runs synchronously inside
   the request-hot path, and re-serializes structures the SDK has to
   re-serialize again moments later. Caching the previous turn's
   `(messageCount, lastMessageHash) → estimate` and only re-estimating
   the delta would cost ~10 lines and convert this to amortized O(Δ).
   For the common "long thread, one new user turn" case the savings are
   real; for the worst case (huge system prompt, every turn) it's the
   difference between a re-stringify per turn and a one-time hash compare.
2. **`Token.estimate` is a heuristic — but the diff treats its output as
   ground truth** for a binary "compact now" decision at 0.85. A
   conservative one-line comment naming the heuristic margin
   (`// estimator under-counts by ~10–15% on tool-call-heavy turns; 0.85
   leaves headroom for that drift`) would explain to the next reader why
   0.85 instead of 0.90/0.95 and make the threshold tunable with intent.
3. **No test covers the new short-circuit arm.** The diff adds a
   user-visible behavior change (a turn can now early-return `"compact"`
   without ever opening the stream) and the "skipped when contextLimit
   is 0" branch is silent fallback that masks misconfiguration. A
   tabletop-style test pinning three cases — `contextLimit=0` ⇒ no
   short-circuit, `tokens < threshold` ⇒ no short-circuit,
   `tokens ≥ threshold` ⇒ short-circuit with `needsCompaction=true` —
   would prevent silent regression of the load-bearing arm.
4. **The new generic patterns at `error.ts:28–29`** (`/max.*tokens.*exceed/i`
   and `/input.*too.*large.*model/i`) are loose; they would also match an
   unrelated provider message like `"max input tokens for tool call must
   not exceed 256"`. Anchoring to common provider phrasing
   (`/(?:max|maximum)\s+(?:context\s+)?tokens?\s+(?:exceeded|exceeds)/i`)
   would close the false-positive path without giving up real coverage.
5. **The new `Token` import path uses the `@/util/token` alias** at
   `processor.ts:13` while neighboring imports
   (`./overflow`, `./session`) are relative — minor but a quick grep
   shows the file already mixes both styles, so this isn't a hard nit.

## Theme tie-in

Move the overflow decision from the *post-stream reactive read site*
(provider error text) to the *pre-stream proactive boundary* (token
estimate against the model's declared limit). Sub-agents that never
reach the post-stream path — and providers that silently truncate
instead of erroring — now get the correct decision at the place that
owns the contract.
