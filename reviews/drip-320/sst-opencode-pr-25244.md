# sst/opencode PR #25244 — fix(app): avoid preview child MCP bootstraps

- Link: https://github.com/sst/opencode/pull/25244
- SHA: `e014c449e6aa298c274c767193fb8ee5f1dcd94c`
- Author: mattgenious
- Stats: +114 / −10, 2 files

## Summary

Fix for #25243. Preview child stores are constructed with `bootstrap: false` (when a project is being previewed via hover/sidebar), but they were still registering directory-scoped queries against `path`, `mcp`, `lsp`, and providers. Those queries would hit backend routes that fail when the child is not a real, fully bootstrapped instance, producing noise and unnecessary work.

The fix gates the `useQueries` factory on the bootstrap flag so non-bootstrapping children never schedule the instance-scoped queries. The companion test file mocks `@tanstack/solid-query` and `@/utils/persist` to validate the gate without spinning up the full app harness.

## Specific references

- `packages/app/src/context/global-sync/child-store.test.ts` L1–L33: introduces a `skipToken` symbol and a hand-rolled mock of `useQueries` that runs `queryFn()` only when it is a real function. This is the right shape — the gate under test is "does the queryFn get assigned to `skipToken` when bootstrap is false?" so the mock must inspect the factory output, not just call it. Looks correct.
- `packages/app/src/context/global-sync/child-store.test.ts` L70–L130: the new `bootstrap false does not load instance-scoped status queries` test asserts that none of `path`, `mcp`, `lsp`, or providers SDK methods are invoked. Good coverage of the four call sites that #25243 fingered.
- The dynamic `import()` of `./child-store` after `mock.module` calls (L36, L48, L72) is required because Bun's `mock.module` must run before the module under test is evaluated. Worth a one-line comment so future contributors don't refactor it back to a top-level import.
- `packages/app/src/context/global-sync/child-store.ts`: not shown in the truncated diff window, but the test names imply the gate is on the `useQueries` factory's `queryFn`. Confirm the gate uses `skipToken` (the canonical solid-query no-op) rather than `enabled: false`, since the test specifically mocks `skipToken` semantics.

## Verdict

verdict: merge-after-nits

## Reasoning

Targeted bug fix backed by a focused unit test. The mock-heavy setup is a smell but unavoidable given the framework. Only nit is documenting the dynamic-import requirement so the test stays correct under future refactors.
