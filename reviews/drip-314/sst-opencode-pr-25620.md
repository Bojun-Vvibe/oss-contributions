# Review — sst/opencode #25620: feat(tui): add input.intercept API for plugin keydown interception

- **PR:** https://github.com/sst/opencode/pull/25620
- **Head SHA:** `05ff633147d7ba2dd3bc87266e1d08777a49c884`
- **Author:** rsdrahat
- **Files:** 6 changed (+42 / −1)
  - `packages/opencode/src/cli/cmd/tui/component/prompt/intercept.ts` (+21 new)
  - `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` (+5)
  - `packages/opencode/src/cli/cmd/tui/plugin/api.tsx` (+6)
  - `packages/opencode/src/cli/cmd/tui/plugin/runtime.ts` (+1)
  - `packages/opencode/test/fixture/tui-plugin.ts` (+3)
  - `packages/plugin/src/tui.ts` (+6/−1)

## Verdict: `merge-after-nits`

## Rationale

This adds a small, well-scoped extension point: plugins can register a keydown handler that runs before the prompt's built-in input handling and short-circuits when it returns truthy. The implementation in `intercept.ts:1-21` is a plain module-level `handlers[]` with a `register()` returning an unregister closure and a `dispatch()` that walks handlers and stops at the first truthy result. Wired in `prompt/index.tsx:1108-1113` between paste detection and clipboard probing — placement looks deliberate so plugins can preempt the textarea's default behaviour without fighting the paste path.

Two nits. First, `intercept.ts` is missing a trailing newline (per the diff `\ No newline at end of file`); minor but consistent style across the package would prefer the newline. Second, the handler list is process-global (a `const handlers: InputInterceptHandler[]`), so multiple plugins are dispatched in registration order with no per-plugin scoping — that's likely fine for this surface but worth documenting in the public type at `packages/plugin/src/tui.ts:480` ("handlers run in registration order; first truthy result wins"). Otherwise plugin authors will discover ordering by surprise.

The fixture update in `test/fixture/tui-plugin.ts:175-177` returns a no-op unregister, which keeps existing tests green but doesn't actually exercise the new dispatch path. A small unit test asserting (a) registration order, (b) truthy-short-circuit, and (c) unregister actually removes the handler would lock in the contract — none of those are exercised here. Not blocking, but the API surface is small enough that adding tests is cheap.

## Banned-string check

Diff scanned; no banned tokens present.
