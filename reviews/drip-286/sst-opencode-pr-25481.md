---
repo: sst/opencode
pr: 25481
head_sha: dc7cff5fd3a499a6fb2231570557cc55864caf13
title: "feat(cli): auto-dispose InstanceContext after effectCmd handlers"
verdict: merge-as-is
reviewed_at: 2026-05-03
---

# Review: sst/opencode#25481 — `feat(cli): auto-dispose InstanceContext after effectCmd handlers`

**Head SHA:** `dc7cff5fd3a499a6fb2231570557cc55864caf13`
**Stat:** +18 / −1 across 1 file (`packages/opencode/src/cli/effect-cmd.ts`).

## What it changes

The `effectCmd` factory previously called `store.provide({ directory }, opts.handler(args))` and let the handler run to completion without an explicit dispose. The new version (effect-cmd.ts:54–63) wraps the handler body in `Effect.ensuring(store.dispose(ctx))`:

```ts
InstanceStore.Service.use((store) =>
  store.provide(
    { directory },
    Effect.gen(function* () {
      const ctx = yield* InstanceRef
      const body = opts.handler(args)
      return ctx ? yield* body.pipe(Effect.ensuring(store.dispose(ctx))) : yield* body
    }),
  ),
)
```

So every `effectCmd` invocation now runs `store.dispose(ctx)` on every Exit
path (success / typed failure / defect / interruption) and emits the
`server.instance.disposed` IPC event — matching legacy `bootstrap()`
finally semantics without per-handler boilerplate.

## Assessment

- This is the right fix for the footgun the PR description calls out
  (the 14× repeated `dispose` pattern flagged by #25479 reviews). Once
  this lands, the explicit `Effect.ensuring(store.dispose(ctx))` calls
  in `import.ts`, `export.ts`, `stats.ts`, and `debug/*.ts` become
  redundant — the PR explicitly defers that cleanup to a follow-up,
  which is correct sequencing (don't strip the explicit calls until
  this auto-dispose ships).
- The `ctx ? ... : yield* body` guard handles the case where
  `InstanceRef` is absent (unloaded store / very early bootstrap) by
  falling through to the unwrapped body. That's the same defensive shape
  the rest of the codebase uses around `InstanceRef`.
- Idempotency claim is load-bearing here: the PR description states
  `disposeEntry` checks `cache.get(directory) !== entry` and returns
  false on the second call, which is what makes the "redundant explicit
  dispose in already-converted commands" not a bug. I didn't re-derive
  that from `instance-store.ts` in this review, but the existing
  converted commands' tests passing (`bun run test test/cli/`, 137 pass)
  is strong evidence the double-dispose is harmless.
- The doc comment (effect-cmd.ts:24–28) is clear about *why* the
  ensuring is there, which matters because the next person to "tidy up"
  this layer would otherwise delete it.

## Nits

- (Non-blocking) The TS narrowing on `ctx` requires the conditional
  yield. A small helper like `withDispose(store, body)` or pushing the
  guard into `store.provide` itself would let the `effectCmd` body stay
  flat. Not in scope here.
- (Non-blocking) No new test specifically asserting the
  `server.instance.disposed` event fires from a failing `effectCmd`
  handler. The 137-test pass is good signal but a 5-line targeted test
  would lock in the contract.

## Verdict

**`merge-as-is`** — minimal, correct fix to a documented footgun;
preserves legacy semantics; sequenced sensibly with the cleanup
follow-up. Ship.
