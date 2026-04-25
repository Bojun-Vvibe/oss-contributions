# anomalyco/opencode#24233 — fix(provider): honor per-model reasoning token pricing

**Verdict: merge-as-is**

- PR: https://github.com/anomalyco/opencode/pull/24233
- Author: willsarg (Will Sarg)
- Base SHA: `1e4b7b5451dc924515444c006f6babbb9f24bc85`
- Head SHA: `faf4025b6fc6c35d68bb5c00c3bb359cab1c6edc`
- +68 / -3

## Summary

Removes a long-standing TODO that hard-coded reasoning tokens to be
billed at the same rate as output tokens. Adds an optional
`reasoning: number` field to the cost schema (both top-level and the
`context_over_200k` overlay) and threads it through the four cost
struct definitions (`config/provider.ts`, `provider/models.ts`,
`provider/provider.ts` x2 — once for the inbound `models.dev` shape
and once for the user-config merge layer). Session cost calculation
in `session/session.ts:354` now multiplies reasoning tokens by
`costInfo?.reasoning ?? costInfo?.output ?? 0` instead of always
`costInfo?.output`. Two new tests cover the explicit-reasoning-price
case and the fallback-to-output case.

## Specific findings

- `packages/opencode/src/session/session.ts:354` (head SHA
  `faf4025b6fc6c35d68bb5c00c3bb359cab1c6edc`):
  ```ts
  // Use the model's explicit reasoning rate when models.dev (or user config)
  // exposes one, otherwise fall back to the output rate as before.
  .add(new Decimal(tokens.reasoning).mul(costInfo?.reasoning ?? costInfo?.output ?? 0).div(1_000_000))
  ```
  The double-`??` chain (`reasoning ?? output ?? 0`) is the right
  fallback ladder: explicit reasoning rate wins, otherwise existing
  behavior (charge at output rate), and only if both are unset does
  cost fall to zero. The previous `tokens.reasoning` * `output ?? 0`
  was the documented placeholder and is preserved as the fallback,
  so any model without explicit reasoning pricing sees zero behavior
  change. Decimal arithmetic semantics (the `Decimal(x).mul(y).div(1M)`
  pattern) are unchanged.
- `packages/opencode/src/provider/provider.ts:855` and `:951` — the
  `cost()` mapper that translates inbound `models.dev` shape to the
  internal `Model["cost"]` shape now copies `reasoning: c?.reasoning`
  at both the top level and inside the `experimentalOver200K`
  overlay. The `?.` chain correctly preserves the optional shape;
  unset upstream stays unset internally.
- `packages/opencode/src/provider/provider.ts:1186` — the user-config
  merge layer now does
  `model?.cost?.reasoning ?? existingModel?.cost?.reasoning`, mirroring
  the same merge pattern as `input` / `output`. This preserves the
  established precedence rule (user override > existing > default).
  The fact that this was added at exactly the same call shape as
  the surrounding lines is a good sign — easy to audit, easy to
  extend later for additional pricing dimensions.
- `packages/opencode/src/config/provider.ts:23` and
  `packages/opencode/src/provider/models.ts:24` — both `Cost`
  Effect-Schema struct definitions get
  `reasoning: Schema.optional(Schema.Number)` added to the top level
  and inside `context_over_200k`. Optional means existing user
  configs and existing `models.dev` payloads stay backward-compatible
  — no migration needed.
- `packages/opencode/test/session/compaction.test.ts:2089` — two new
  tests. The explicit case asserts
  `(800*0.6 + 20*4 + 100*1 + 200*0.15) / 1_000_000` with a
  `reasoning: 1` rate cleanly distinct from the `output: 4` rate, so
  any regression conflating the two would fail loudly. The fallback
  case asserts `30 * 4` for reasoning tokens when no `reasoning`
  field is set, locking in the backward-compat path.

## Rationale

This is the textbook shape for retiring a TODO without breaking
existing behavior: introduce an optional new field, default it to
the previous behavior when absent, and add tests for both branches.
The previous code carried a literal "TODO: update models.dev to have
better pricing model, for now: charge reasoning tokens at the same
rate as output tokens" — which billed users incorrectly for any
provider where reasoning is priced differently from output (notably
the GPT-5/o-series families where reasoning has a distinct rate).
The four touchpoints (config schema, internal models schema, the
inbound `cost()` mapper, the user-config merge) form a complete
chain — there's no orphan field that gets parsed but ignored, and
no parsing surface that drops the field on the floor. The
`reasoning ?? output ?? 0` fallback ladder in the session cost
calc is the correct shape: it preserves the previous behavior for
every model that doesn't yet expose a `reasoning` rate (so no user
sees a sudden cost change) while letting any provider entry that
*does* set the field opt into accurate pricing immediately. The two
tests are well-chosen — one with explicit reasoning rate distinct
from output (1 vs 4), one with the fallback path — and the
arithmetic in the assertion (`800*0.6 + 20*4 + 100*1 + 200*0.15`)
is hand-verifiable, so a future regression wouldn't get hidden by
floating-point fuzz. No nits. Merge.
