# Review: sst/opencode #25258 — fix: ensure MiMo models display correct context limit in tooltip

- PR: https://github.com/sst/opencode/pull/25258
- Head SHA: `e22c0cea47d57ce9adfb08c9f2c9b5ed9f5afb0b`
- Author: ShadyUnderLight
- Size: +8 / -4

## Summary

Two-file defensive fix for a tooltip bug where Xiaomi MiMo models surface
"Context limit 0" instead of the real 256K–1M token figures. The root
cause is that `models.dev` does not always emit a `limit.context` value
for MiMo entries, and the consumer side blindly dereferences
`model.limit.context`. The PR plugs both holes: optional chaining at the
display site so the tooltip simply omits the line when no value is
known, and an `?? 0` / `?.` shape on the provider mapper so missing
fields don't crash the pipeline upstream.

## Specific citations

- `packages/app/src/components/model-tooltip.tsx:9-14` — replaces the
  one-liner `language.t("model.tooltip.context", { limit: ... })` with
  a guarded variant that returns an empty string when the limit is
  `undefined`/`null`. This is the right UI shape: better to render
  nothing than "Context limit 0", which actively misleads.
- `packages/opencode/src/provider/provider.ts:1002-1009` (mapped from
  diff lines 22-32 of `fromModelsDevModel`) — the structural fix.
  `model.limit?.context ?? 0` handles the case where the upstream
  payload omits the `limit` object entirely. Note the asymmetry: the
  PR uses `?? 0` for `context` and `output` but leaves `input` as
  `model.limit?.input` (no fallback). That mirrors how those fields
  are typed downstream (`output` is treated as required, `input` as
  optional), so the asymmetry is deliberate, but it would be worth a
  one-line comment.

## Verdict

**merge-after-nits**

## Rationale

The fix is small, surgical, and addresses a visible regression. The
two changes work together — the provider mapper guarantees a numeric
value (or `undefined`), the tooltip handles `undefined` gracefully —
which is the correct defense-in-depth posture for a data-derived
display string.

The one structural concern is that `?? 0` at the provider layer
silently coerces "unknown" into "zero context" for any downstream
consumer that doesn't optional-chain. The tooltip is now safe, but
anything else reading `model.limit.context` (e.g. a token-budget
calculator, a model picker that ranks by context, the request-time
truncator) will now see a 0 instead of `undefined` and behave
incorrectly. A safer fix would be to leave the provider layer as
`model.limit?.context` (preserving `undefined`) and rely on every
consumer to optional-chain — but that's a larger refactor and out of
scope for a tooltip bugfix.

The provider-layer asymmetry (`output: ?? 0`, `input: no fallback`)
should be either documented inline or made consistent.

No tests added. For a 4-line change to two known-broken display paths
this is acceptable, but a snapshot test for `fromModelsDevModel` with
a `limit`-less input would catch any regression and would be ~10
lines of fixture.

## Nits

1. Add inline comment explaining the `?? 0` choice (display sentinel
   vs. real value) and why `input` is treated differently.
2. Add a unit test for `fromModelsDevModel` with an input model that
   lacks a `limit` field altogether.
3. Consider matching the tooltip's "omit when missing" pattern at the
   provider level — emit `undefined` and let consumers branch — to
   avoid the silent-zero footgun.
