# charmbracelet/crush PR #2498 — replace sticky LSP unavailable cache with retry backoff

Link: https://github.com/charmbracelet/crush/pull/2498
State: MERGED · Author: iceymoss · +73 / −14 · Files: 2

## Summary
Removes the package-level `var unavailable = csync.NewMap[string, struct{}]()` that previously
recorded LSP servers whose binaries failed `exec.LookPath`, and replaces it with per-`Manager`
state plus a 30-second retry window. New helpers `recentlyUnavailable`, `markUnavailable`,
`clearUnavailable` mediate the access. `now` is now an injectable `func() time.Time` for
testability. Skip behavior for the `skipAutoStartCommands` set is unchanged.

## Strengths
- Fixes a real, observable bug: a process that started before the user installed `gopls`
  permanently never auto-starts gopls until restart. The new 30-second backoff is short enough
  to feel responsive and long enough to avoid hammering `LookPath` on a hot path.
- Per-`Manager` state eliminates a lurking test-isolation hazard — the package global made
  every test run share state, with all the usual flake risks.
- Injectable `now` cleanly enables time-controlled tests, and the new `TestUnavailableBackoff`
  exercises the three meaningful states (immediate, post-window, post-clear).
- `clearUnavailable` is called on every successful `LookPath`, so binaries installed mid-session
  recover automatically without waiting for the 30s timer.

## Concerns / Risks
- **Concurrency window: read-then-mutate.** `recentlyUnavailable` does `Get` → conditional `Del`
  without a lock. Two goroutines arriving at the boundary of the 30s window can both observe
  "expired", both delete, and both proceed to `LookPath` — fine in practice (idempotent), but
  worth noting that `csync.Map` does not provide compare-and-swap semantics here. A comment to
  that effect would prevent a future reader from "fixing" the race in a way that introduces
  serialization.
- **Backoff is not jittered.** If a workspace declares many auto-startable LSP servers and
  none of them are installed, every cycle will retry all of them in a synchronized burst every
  30 seconds. Low-cost in absolute terms, but a small jitter per server name would smooth the
  load.
- **Test does not exercise the actual `startServer` path.** The new test pokes `Manager`
  fields directly. A higher-level test that calls `startServer` with a deliberately missing
  command and asserts the second call within 30s does not invoke `LookPath` would catch a
  refactor that forgets to consult `recentlyUnavailable`. Right now the integration of the
  helper into the auto-start path is unguarded.
- **`isUserConfigured` bypasses the backoff entirely**, which is intentional per the PR
  description, but means a user-configured server with a missing binary will hammer
  `LookPath` on every request. Worth a separate slow-path log so users notice their config
  references a non-existent binary instead of silently never getting LSP.
- **Edge case: `now()` going backward.** `s.now()` is a `func()`, mostly `time.Now`. With
  monotonic clocks this is fine, but a test with a custom `now` that goes backward (e.g.
  fixture replay) would never expire the backoff. Not a real-world bug, just a robustness
  observation.

## Suggested follow-ups
1. Add an integration test that drives `startServer` and asserts `LookPath` is not called
   inside the 30s window.
2. Consider per-name jitter (±5s) to avoid thundering-herd retries when many LSPs are missing.
3. Surface a one-shot warning when a *user-configured* LSP command is missing; the silent
   bypass of the backoff makes config typos hard to notice.
