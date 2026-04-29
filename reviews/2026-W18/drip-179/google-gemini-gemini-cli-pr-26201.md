# google-gemini/gemini-cli#26201 — fix(core): clamp remaining context tokens

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26201
- **Head SHA**: `c26ea8dc9ef68d2f374f812c0b2c22212212f758`
- **Size**: +39 / −2, 2 files
- **Files**: `packages/core/src/core/client.ts` (+4/-2), `packages/core/src/core/client.test.ts` (+35/-0)
- **Fixes**: #26188

## Context

In `GeminiClient.sendMessageStream`, when the *current prompt* already exceeds the model's token limit (e.g., 1200 tokens against a 1000-token cap), the `ContextWindowWillOverflow` event was being emitted with a *negative* `remainingTokenCount` like `-200`. The CLI surfaces that count to the user, so they'd see a confusing message like "you have -200 tokens remaining". The fix clamps the value at zero.

## Why this exists / problem solved

The arithmetic at `client.ts:646-647` (pre-PR) was:

```ts
const remainingTokenCount =
  tokenLimit(modelForLimitCheck) - this.getChat().getLastPromptTokenCount();
```

Subtraction of two `number`s where the right operand can exceed the left has the obvious failure mode. The user-facing channel is the `ContextWindowWillOverflow` event payload's `remainingTokenCount` field, which downstream UI surfaces ("you have N tokens remaining before overflow"). A negative value is meaningless to the user — "remaining" can't go below zero.

The PR's fix at `:646-649` is a one-liner:

```ts
const remainingTokenCount = Math.max(
  0,
  tokenLimit(modelForLimitCheck) - this.getChat().getLastPromptTokenCount(),
);
```

Plus a 35-line test at `client.test.ts:1555-1589` covering the exact reproduction case (1200-token prompt, 1000-token limit, asserts emitted event has `remainingTokenCount: 0` and `estimatedRequestTokenCount: 3`).

## Design analysis

### The fix itself

`Math.max(0, x)` is the textbook clamp-to-floor idiom in JS. It's correct, readable, and zero-allocation. The two-arg shape is preferred over `x < 0 ? 0 : x` for readability and over `Math.max(0, Number.isFinite(x) ? x : 0)` for terseness — `Math.max` already returns `NaN` for non-finite operands which is its own bug class, but `tokenLimit()` returns a hard-coded number per model and `getLastPromptTokenCount()` returns the cached last-known count (also a number), so neither operand can be `NaN`/`Infinity` in practice. Safe.

The test at `client.test.ts:1555-1589` is well-shaped:

- Mocks `tokenLimit` to return `1000` so the test is independent of any specific model's real limit.
- Sets `getLastPromptTokenCount` to `1200`, plus `tryCompressChat` returns `NOOP` (so we don't accidentally take the compression branch and skip the overflow event).
- Sends `[{ text: 'short request' }]` — small enough that `estimatedRequestTokenCount` is 3 (the `'short request'` text), confirming the *current prompt* size, not the *historical* size, drives the calculation.
- Asserts the event payload exactly: `{type: ContextWindowWillOverflow, value: {estimatedRequestTokenCount: 3, remainingTokenCount: 0}}`.

The exact-equality on `estimatedRequestTokenCount: 3` is interesting — it pins the heuristic. If the heuristic ever changes (CHARS_PER_TOKEN drift, whitespace tokenization, etc.), the test will fail with a clear message and a new pin. Acceptable; some teams prefer `>= 0 && <= 10` for hand-wavy heuristics, but the exact pin doubles as documentation of the heuristic's current behavior.

### What this PR doesn't fix

The PR is intentionally narrow — it clamps the *display* value but doesn't change the *decision* logic that emits `ContextWindowWillOverflow`. From the surrounding diff context, the event is still emitted whenever the request would overflow; what changes is just the displayed `remainingTokenCount` field. This is correct: the overflow signal is still accurate ("you will overflow"), but the magnitude field no longer goes negative.

What this *doesn't* address — and which the issue #26188 may or may not have asked for — is the question "what should `estimatedRequestTokenCount` show when the *current request* fits but the *cumulative history + request* overflows?" The PR's test covers the case where the current prompt token count alone exceeds the limit (`getLastPromptTokenCount: 1200` against `tokenLimit: 1000`). The naming `getLastPromptTokenCount` suggests this is the "size of the last assembled prompt sent to the model," which already includes history. So the test's `'short request'` is the *next* user message, but the `1200` is the *prior assembled prompt*. The math `tokenLimit - getLastPromptTokenCount` is therefore "what's left after the last turn was assembled," and clamping that at zero is the right semantics.

### Naming/clarity nit

The variable name `modelForLimitCheck` at `:643` and `:646` (just outside the diff window) implies there's some "sticky model" logic that selects which model's limit to check (the existing test "should use the sticky model's token limit for the overflow check" at `:1591` confirms this). The PR doesn't touch that logic — good, it's outside the fix's scope.

`remainingTokenCount` itself is a slightly misleading name now that it's clamped — it should perhaps be `displayableRemainingTokenCount` or come with a JSDoc comment noting "clamped at zero; the true raw value can be negative when the current prompt already exceeds the limit." A reader wondering "why does my dashboard never show negative remaining" would benefit from that pointer. Non-blocking.

## Risks

- **Downstream consumers depending on negative values for "by how much did we overflow."** If any UI or telemetry code reads `remainingTokenCount` and uses its negativity as a "we're over by X" signal, they'll silently lose that signal after this fix. A `git grep "remainingTokenCount"` across the repo would surface any such consumer. The PR doesn't show this check was done. Probably none exists (the field is described as "remaining," not "delta") but worth a 30-second sweep.
- **Telemetry continuity.** If any telemetry pipeline aggregates `remainingTokenCount` as `min` or `mean`, the distribution will shift after this fix because all previously-negative values are now zero. Not a regression in correctness, just a discontinuity in the time-series.
- **Test brittleness around `estimatedRequestTokenCount: 3`.** As noted above, exact pin to `3` ties the test to the current heuristic. Acceptable.
- **The clamp masks the underlying overflow magnitude from logs.** If `client.ts` ever logs the raw `tokenLimit - getLastPromptTokenCount` for diagnostic purposes (e.g., "overflow by 200 tokens"), that visibility is lost after the clamp. Currently the diff shows only one consumer of the value (the event emit), so no log impact today, but if a future "log how badly we overflowed" diagnostic is added, the clamp will hide the magnitude.

## Suggestions

1. **Grep the repo for `remainingTokenCount` consumers** before merge to confirm no UI code uses negative values as a "by how much" signal. 30-second check; if anything turns up, decide whether to also expose a separate `overflowAmount` field. (Rough check: `rg 'remainingTokenCount' packages/`.)
2. **Add a one-line JSDoc** at the variable declaration documenting the clamp:
   ```ts
   // Clamped at zero — `getLastPromptTokenCount()` can exceed `tokenLimit` when
   // the assembled prompt already overflows; surfacing a negative value is
   // user-confusing.
   const remainingTokenCount = Math.max(0, ...);
   ```
3. **Consider also emitting the raw delta as a separate field** (`overflowByTokenCount: Math.max(0, getLastPromptTokenCount - tokenLimit)`) so dashboards/telemetry that *want* the overflow magnitude have a non-clamped signal. Optional; do this only if a real consumer requests it.
4. **Add a parametric test** covering: the original "negative" case (1200 vs 1000 → 0), the boundary case (1000 vs 1000 → 0), the comfortable case (500 vs 1000 → 500). The PR has only the negative case; the other two are good regression-fence cases. Five extra lines.
5. **Tag the test name with the issue number** — `it('should not report negative remaining tokens when the current prompt already exceeds the limit (#26188)')` — so future grep-by-issue is easy. Stylistic; some projects prefer this in the comment instead.

## Verdict

**merge-as-is** — the fix is one line, mechanically correct, exercises the right test surface, and the only meaningful concern (downstream consumers of negative values) is almost certainly empty. The JSDoc-comment and parametric-test suggestions are improvements but not blockers. Ship it.

## What I learned

The smallest possible fix for "user sees a confusing negative number" is `Math.max(0, x)` on the display value, *not* "rework the whole token-accounting subsystem so the underlying number can never go negative." The latter would touch dozens of call sites for a one-line user-facing problem. This PR demonstrates the right scope-discipline: identify the exact display surface, clamp there, leave the underlying calculation honest. The corollary is that whenever you do clamp at the display layer, you should add a JSDoc note pointing future readers at the underlying-truth source — otherwise five years from now someone debugging "why does our overflow gauge cap at zero" has to re-derive the clamp's existence from `git blame`. The 1-line fix and the 1-line comment are both required to leave the codebase honestly explained.
