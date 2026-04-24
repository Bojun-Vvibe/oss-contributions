# PR #2698 — test(hyper): add provider env behavior coverage

**Repo:** charmbracelet/crush  
**Surface:** new file `internal/agent/hyper/provider_test.go` (+96)  
**Severity:** Low (test-only PR; quality-of-tests review)

## What changed

Adds Go test coverage for three `sync.OnceValue`-wrapped package-level
functions in the hyper provider:

- `Enabled()` — env-var precedence and bool parsing across `HYPER`,
  `HYPERCRUSH`, `HYPER_ENABLE`, `HYPER_ENABLED`.
- `BaseURL()` — `HYPER_URL` override / default fallback.
- `Embedded()` — endpoint rewrite to `<HYPER_URL>/api/v1/fantasy` when
  the env var is set.

The tests use `t.Setenv(...)` plus locally-defined `resetXxxForTest()`
helpers that *replace* the package-level `sync.OnceValue` wrappers with
fresh instances, since `sync.OnceValue` caches the result of its first
call and would otherwise pin the value across tests.

## What's risky / wrong / missing

1. **`resetXxxForTest()` mutates package-level state from a test file.**
   This is the only way to defeat `sync.OnceValue` caching, but it has
   real consequences: any test in the same package running concurrently
   (`go test -parallel`) — or any test that imports this package
   indirectly — will observe the mutated `Enabled`/`BaseURL`/`Embedded`
   slot. The PR does not call `t.Parallel()` and does not document this
   constraint. Add a `// NOT SAFE FOR PARALLEL` comment at minimum, or
   wrap the resets in a `sync.Mutex` shared with any other test that
   touches these.

2. **Test bodies duplicate production logic.** Each `resetXxxForTest()`
   re-implements the cmp.Or chain that lives in production code. If the
   production order ever changes (e.g. `HYPER_ENABLE` is dropped, or a
   new `HYPER_OFF` kill-switch is added), the test will keep passing
   against the stale logic and silently lose coverage. Better: expose a
   test-only `resetForTest()` from the production package so the env
   resolution lives in exactly one place.

3. **Negative case for invalid bool is half-tested.** The "HYPER has
   highest priority and invalid bool resolves false" assertion is
   correct, but `cmp.Or` returns the *first non-zero* argument — so
   `HYPER="not-a-bool"` short-circuits before `HYPERCRUSH="true"` is
   even consulted. The test is asserting the right thing but for the
   wrong reason; renaming the test or adding a comment would clarify.

4. **`TestEmbedded` does not assert what `APIEndpoint` was actually
   rewritten to** in the truncated diff (file ends mid-test in my
   diff window). Confirm the assertion is
   `require.Equal(t, "https://proxy.example.com/api/v1/fantasy", p.APIEndpoint)`
   — and not just `require.NotEmpty`.

5. **No teardown.** `t.Setenv` automatically restores the env at end of
   test, but the swapped `sync.OnceValue` wrappers are *not* restored.
   If any later test in the package calls `Enabled()` directly, it now
   sees whichever wrapper this file installed last. Add a
   `t.Cleanup(func() { resetEnabledForTest() /* with empty env */ })`
   to each test, or move the resets into a helper that registers cleanup.

## Suggested fix

Move the reset helpers into the production package guarded by
`//go:build test` or a non-exported `resetForTest_xxx` function, add
`t.Cleanup` for each swap, and explicitly forbid `t.Parallel()` with a
comment.

## Severity

Low. It's net-positive coverage — the env-precedence behavior was
previously untested. The structural concerns above will bite later if
this package grows more once-cached state.
