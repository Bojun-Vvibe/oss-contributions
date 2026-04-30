# sst/opencode #25042 — test: cover ConfigService helper

- **URL:** https://github.com/sst/opencode/pull/25042
- **Head SHA:** `556a5b2356c1260c0c6fea9de7a9d0fe7fd11983`
- **Merge SHA:** `c49bf0b402d54c453e8bfd39cce465bee9281a43`
- **Files:** `packages/opencode/test/effect/config-service.test.ts` (new, +65)
- **Verdict:** `merge-as-is`

## What changed

Adds a focused unit suite for the `ConfigService.Service<Self>()(id, fields)` factory that landed alongside #25035. Three `it.effect` cases at `config-service.test.ts:18-64` cover the three semantically distinct entry points of the factory:

1. **`defaultLayer parses values from the active ConfigProvider`** at `:18-32` — provides `ConfigProvider.fromUnknown({ NAME, TOKEN, PORT })`, then asserts `Option.some("secret")` for the `Config.option`-wrapped field and `4096` for the `Config.number(...).pipe(Config.withDefault(3000))`-wrapped field after string→number coercion.
2. **`defaultLayer applies Effect Config defaults`** at `:34-43` — provides only `NAME`, asserts `token` is `Option.none()` (the `Config.option` "absent" case) and `port` is `3000` (the `Config.withDefault` fallback). Locks the asymmetry between `Config.option` and `Config.withDefault` semantics.
3. **`layer provides an already parsed service value`** at `:45-63` — bypasses `ConfigProvider` entirely and feeds a typed object straight into `TestConfig.layer({...})`, asserting structural equality with `satisfies Context.Service.Shape<typeof TestConfig>` to lock the inferred shape against the schema.

`fromConfig` at `:12-13` is the shared helper composing `TestConfig.defaultLayer.pipe(Layer.provide(ConfigProvider.layer(ConfigProvider.fromUnknown(input))))` — the canonical Effect "swap the active provider for tests" pattern, expressed once.

## Why it's right

- **Tests the contract, not the implementation.** The three cases correspond exactly to the three `Self`-returning entry points the factory exposes (`defaultLayer` parse-from-provider, `defaultLayer` use-defaults, and `layer(input)` direct-value). Anyone refactoring the factory has a green/red signal for each entry point in isolation, and a single failure tells you which contract regressed.
- **`satisfies Context.Service.Shape<typeof TestConfig>` at `:60` is the strongest assertion in the file.** It locks the public type *shape* of the service against the field schema — if a future change adds a field to the factory but forgets to thread it through the inferred `Self`, the third test fails at type-check time, not at runtime.
- **`Config.string("TOKEN").pipe(Config.option)` and `Config.number("PORT").pipe(Config.withDefault(3000))` at `:7-8` mirror the actual production usage in `authorization.ts:24-25` from #25035** — so this isn't a synthetic test of "any factory works", it's a test that the *real* combinator stacks the project uses keep working through the factory wrapper.
- **No mocks, no spies, no sleep, no globals.** Pure Effect. The whole file is composed of `Effect.provide(...)` calls and value assertions. That's the right shape for a layer-injection-friendly module.

## No nits

The file is 65 lines, hits all three contract entry points, asserts both happy-path and default-path semantics, and locks the inferred shape via a `satisfies` clause. Nothing to add.
